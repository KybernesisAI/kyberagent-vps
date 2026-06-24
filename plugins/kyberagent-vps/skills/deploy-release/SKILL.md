---
name: deploy-release
description: Deploy an official tagged KyberAgent release (e.g. 0.1.10-alpha) to an existing VPS — check out the release tag, rebuild the host daemon/CLI/Live Artifacts/Open Design on the box, restart the services, and pull the matching GHCR environment image for Docker work-envs. Use when the user asks to "update the VPS to the latest release", "deploy 0.1.x-alpha to the box", "bump the server to the new release and env image", or "push the tagged release to the VPS". For ad-hoc dev-file syncs that aren't a tagged release, use update-build instead.
allowed-tools: Bash, Read
---

# Deploy a tagged release to a KyberAgent VPS

Releases are cut as git tags (`vX.Y.Z-alpha`) in the **private
KBDE-KyberAgent-Enterprise** repo. The public **KBDE-KyberAgent-Releases** repo
carries only the **desktop** auto-update feed (signed `.dmg`/`.zip` +
`latest-mac.yml`) — *no daemon source there*. A matching **environment image** is
published to GHCR per tag (`ghcr.io/kybernesisai/kyberagent-env:<tag>`).

So a VPS release deploy is **two things at one tag**:
1. Rebuild the **host daemon/CLI/Live Artifacts/Open Design** from the tagged
   Enterprise source — the box runs the esbuild bundle
   (`ExecStart=node …/packages/daemon/dist/kyberagent-daemon.mjs`), and
   `/opt/kyberagent/app` is a plain copy (not a git checkout), so we rsync source
   and build **on the box** (Linux-native deps).
2. Pull the **env image** at the same tag (for Docker work-env containers).

## Prerequisites
- A local **KBDE-KyberAgent-Enterprise** checkout (the private dev repo — releases
  are cut here and it's the source of truth for the VPS).
- Root SSH to the VPS; the daemon bearer is read off the box.
- The desktop's **Open Design payload** built into the checkout
  (`apps/desktop/resources/apps/daemon/{dist,node_modules}`) — the host ships it.

## 1. Connect + choose the version
```bash
VPS="$(cat ~/.config/kyberagent-vps/host 2>/dev/null)"
# ↑ saved host; if empty, ask for the box's IP / Tailscale name and save it:
#   mkdir -p ~/.config/kyberagent-vps && printf %s "<host>" > ~/.config/kyberagent-vps/host
kssh() { ssh -o ConnectTimeout=20 -o StrictHostKeyChecking=accept-new root@"$VPS" "$@"; }
BEARER=$(kssh "grep '^KYBERAGENT_DAEMON_TOKEN=' /etc/kyberagent/remote.env | cut -d= -f2-")

VERSION=<x.y.z-alpha>     # e.g. 0.1.10-alpha. Omit to take the latest published release:
#   VERSION=$(gh release view -R KybernesisAI/KBDE-KyberAgent-Releases --json tagName -q .tagName | sed 's/^v//')
# confirm the tag exists in the public release feed:
gh release view "v$VERSION" -R KybernesisAI/KBDE-KyberAgent-Releases --json tagName -q .tagName
```

## 2. Check out the release in your local Enterprise checkout
```bash
REPO=/path/to/KBDE-KyberAgent-Enterprise
git -C "$REPO" fetch --tags
git -C "$REPO" checkout "v$VERSION"
# preflight: the Open Design payload must be present (the host daemon ships it):
test -d "$REPO/apps/desktop/resources/apps/daemon/dist" \
  && test -d "$REPO/apps/desktop/resources/apps/daemon/node_modules" \
  || echo "✗ OD payload missing — build the desktop first (see the deploy skill)"
```

## 3. Sync the release source to the box
> **Never re-run `install.sh` for an update.** The installer regenerates the
> daemon / Open Design / Live Artifacts **tokens**, which breaks every paired
> client (this is what caused an OD bearer drift before). Sync source + rebuild
> in place instead.
```bash
# packages source (delete stale; keep the box's node_modules + dist + tsbuildinfo):
rsync -az --delete --exclude node_modules --exclude dist --exclude '*.tsbuildinfo' \
  -e "ssh -o ConnectTimeout=20" "$REPO/packages/" root@"$VPS":/opt/kyberagent/app/packages/
# root build-config — REQUIRED so `pnpm install` resolves the release's deps:
printf '%s\n' package.json pnpm-lock.yaml pnpm-workspace.yaml .npmrc tsconfig.base.json | \
  rsync -az --files-from=- -e "ssh -o ConnectTimeout=20" "$REPO/" root@"$VPS":/opt/kyberagent/app/
```
> Open Design is a **separate bundled payload** (`apps/desktop/resources/apps/daemon`).
> A normal release leaves it as-is — only re-ship + rebuild it if the release
> actually changed OD (and remember its `node_modules`/`dist` are needed).

## 4. Rebuild on the box (Linux-native) + restart
```bash
kssh "chown -R KyberAgent:KyberAgent /opt/kyberagent/app/packages \
  /opt/kyberagent/app/package.json /opt/kyberagent/app/pnpm-lock.yaml \
  /opt/kyberagent/app/pnpm-workspace.yaml /opt/kyberagent/app/.npmrc /opt/kyberagent/app/tsconfig.base.json"
kssh "sudo -iu KyberAgent bash -lc 'cd /opt/kyberagent/app \
  && pnpm install \
  && pnpm --filter @kyberagent/daemon build:bundle \
  && pnpm --filter @kyberagent/cli build \
  && pnpm --filter @kyberagent/live-artifacts-daemon build'"
kssh "systemctl restart kyberagent-daemon kyberagent-live-artifacts"
```
> - **Scoped builds only.** `build:bundle` is esbuild bundling straight from
>   source — do **NOT** run `pnpm -r build` (it would try to build the Electron
>   desktop on the box and fail). Build the daemon bundle + CLI + Live Artifacts only.
> - `pnpm install` recompiles native deps (`better-sqlite3`, `sqlite-vec`, `@libsql`)
>   for Linux; a harmless `prepare: not in a git directory` warning is expected
>   (`/opt/kyberagent/app` isn't a git repo).
> - The daemon restart bounces all agents for a few seconds. **Open Design is not
>   rebuilt here, so don't restart it.** Confirm the new bundle by its size/mtime
>   (`ls …/dist/kyberagent-daemon.mjs`) — the `/health` `version` field stays a
>   hardcoded `0.1.0-alpha` and does **not** reflect the release.

## 5. Pull the matching environment image (Docker work-envs)
```bash
kssh "docker pull ghcr.io/kybernesisai/kyberagent-env:$VERSION \
  && docker tag ghcr.io/kybernesisai/kyberagent-env:$VERSION kyberagent-env:latest"
# If KYBERAGENT_ENV_IMAGE is pinned in /etc/kyberagent/remote.env, bump it to :$VERSION
# and restart the daemon so new work-envs use the release image.
```

## 6. Verify
```bash
kssh "systemctl is-active kyberagent-daemon kyberagent-live-artifacts open-design-daemon"
kssh "curl -s -H 'Authorization: Bearer $BEARER' http://127.0.0.1:8765/health"
kssh "curl -s -H 'Authorization: Bearer $BEARER' http://127.0.0.1:8765/environments/preflight"   # imagePresent:true, ready:true
```
Then run the **health-check** skill for a full sweep. Agent config (Claude login,
Plaud/OD bearers) lives in files that the rebuild doesn't touch — but verify:
if Claude reads "logged out" at inference after the bump, re-run the Claude
device-flow login in tmux (same as the deploy skill's login step).

## Notes
- **Why build on the box, not ship a bundle:** native deps must match the VPS's
  Linux/libc; building there guarantees it. (`/opt/kyberagent/app` being a plain
  copy is also why we rsync rather than `git pull`. Converting it to a checkout of
  the Enterprise repo at the tag would let future deploys `git fetch && git checkout v$VERSION` instead.)
- **Fast daemon-only path (no full build):** the published env image already
  contains the built daemon bundle at `/daemon/kyberagent-daemon.mjs`; you *can*
  `docker create` + `docker cp` it onto the host instead of building. It skips
  Live Artifacts + Open Design, so only use it for a daemon-only hotfix, not a
  full release.
- **Rollback:** re-run this skill with the previous `$VERSION` (the env image tags
  for prior releases stay on GHCR; the source tag is in the Enterprise repo).
