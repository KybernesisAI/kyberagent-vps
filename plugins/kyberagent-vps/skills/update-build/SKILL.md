---
name: update-build
description: Deploy a new KyberAgent release to an existing VPS — sync updated source from a local checkout, rebuild the affected artifact (daemon esbuild bundle vs CLI dist vs Live Artifacts), and restart the right services. Use when the user asks to "update the VPS to the new build", "deploy the new release to the box", "push the latest code to the server", or "rebuild the daemon on the VPS".
allowed-tools: Bash, Read
---

# Update the build on a KyberAgent VPS

The box builds from source. Sync the new source from a local
**KBDE-KyberAgent-Enterprise** checkout, rebuild what changed, restart. See
`../../reference/connection.md`.

> **Match the rebuild to what changed** (repo CLAUDE.md): the daemon runs the
> esbuild **bundle**, so daemon source needs `build:bundle`; the `kyberagent` CLI
> runs `dist/index.js`, so CLI source needs `@kyberagent/cli build`. A bare
> `systemctl restart` picks up **neither** without the rebuild.

## 1. Connect
```bash
VPS="$(cat ~/.config/kyberagent-vps/host 2>/dev/null)"
# ↑ The saved host. If it is NON-EMPTY, USE IT and continue — do NOT ask the user.
# ONLY if it is empty: ask for the box's Tailscale name or IP (from the `deploy`
# output, the cloud dashboard, or the Tailscale admin), then save it so nothing
# asks again:  mkdir -p ~/.config/kyberagent-vps && printf %s "<host>" > ~/.config/kyberagent-vps/host
# ("Permission denied" from kssh = your SSH key isn't root-authorized on the box.)
kssh() { ssh -o ConnectTimeout=20 -o StrictHostKeyChecking=accept-new root@"$VPS" "$@"; }
BEARER=$(kssh "grep '^KYBERAGENT_DAEMON_TOKEN=' /etc/kyberagent/remote.env | cut -d= -f2-")
cd /path/to/KBDE-KyberAgent-Enterprise        # local checkout at the new release
```

## 2. Sync source
Per-file (spell out full dest paths — multi-source-to-dir flattens basenames):
```bash
rsync -az -e "ssh -o ConnectTimeout=20" \
  packages/daemon/src/<file>.ts root@$VPS:/opt/kyberagent/app/packages/daemon/src/<file>.ts
```
Whole package for a full release (exclude build output):
```bash
rsync -az --delete -e "ssh" --exclude node_modules --exclude dist \
  packages/daemon/src/ root@$VPS:/opt/kyberagent/app/packages/daemon/src/
```
> If `/opt/kyberagent/app` is a git checkout (`kssh "git -C /opt/kyberagent/app rev-parse --git-dir"` succeeds), `git fetch && git reset --hard <ref>` instead.

## 3. Rebuild AS the app user (fix ownership of anything copied as root first)
```bash
kssh "chown -R KyberAgent:KyberAgent /opt/kyberagent/app/packages"
kssh "sudo -iu KyberAgent bash -lc 'cd /opt/kyberagent/app && pnpm install && pnpm --filter @kyberagent/daemon build:bundle'"
#   CLI changed too?       add:  pnpm --filter @kyberagent/cli build
#   Live Artifacts changed? add: pnpm --filter @kyberagent/live-artifacts-daemon build
```

## 4. Restart + verify
```bash
kssh "systemctl restart kyberagent-daemon"     # + kyberagent-live-artifacts / open-design-daemon if those changed
kssh "systemctl is-active kyberagent-daemon"
kssh "curl -s -H 'Authorization: Bearer $BEARER' http://127.0.0.1:8765/health"
```
Then run the **health-check** skill for a full post-update sweep.
