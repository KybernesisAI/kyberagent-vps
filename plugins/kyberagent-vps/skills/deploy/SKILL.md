---
name: deploy
description: Stand up a complete, production KyberAgent stack on a fresh cloud VPS (DigitalOcean or Hetzner) so a NAMED agent runs 24/7 with its own brain, reachable from the desktop over Tailscale, and able to spin up Dockerized GitHub work-environments. Use when a user asks to "deploy kyberagent to a VPS", "set up a cloud agent", or stand up an always-on remote agent. Handles droplet access + hardening, Node/Tailscale, the host-native install (daemon + CLI + Live Artifacts + Open Design), CREATING the user's agent, desktop pairing values, Claude login, firewall, Docker + GitHub env support, and a full end-to-end eval that ends cleanly. For day-2 operations on an already-deployed box, use the other kyberagent-vps skills instead.
allowed-tools: Bash, Read, Write, Edit
---

# Deploy a KyberAgent cloud agent (end-to-end runbook)

This is the exact, reproducible procedure to stand up an always-on cloud agent.
Run it from a machine that has a **KBDE-KyberAgent-Enterprise checkout** (the
installer builds the daemon from source on the box).

> The desktop app already has a local **orchestrator**; on the VPS you almost
> always want a **separate, named agent** (e.g. `ava`) — not orchestrator. The
> agent-creation step below is mandatory, not optional.

---

## Inputs to collect from the user (ask up front)

1. **Agent name** — a single lowercase word (`^[a-z][a-z0-9-]{1,30}$`), e.g. `ava`.
2. **Server**: either a **DigitalOcean API token** (you create the droplet) OR
   an already-created droplet's **public IP** (they made it via the guide).
3. **Tailscale auth key** — from https://login.tailscale.com/admin/settings/keys
   (Generate auth key; reusable, non-ephemeral).
4. Confirm they have a **Claude Pro/Max** subscription (the agent signs in via
   device flow) and, if they want code envs, will authorize **GitHub** later.

Provider note: **DigitalOcean** (official MCP: `claude mcp add digitalocean`) or
**Hetzner** (≈2–3× cheaper, full API) — both KVM, both run Docker. Plan: **4
vCPU / 8 GB / 160 GB**, image **"Docker on Ubuntu 22.04"**.

Throughout, use shell variables so nothing is hardcoded:
```bash
IP=<droplet-public-ip>; AGENT=<agent-name>; TS_KEY=<tailscale-auth-key>
SSH_OPTS="-o ConnectTimeout=20 -o StrictHostKeyChecking=accept-new"
```

---

## 0. SSH access (resolve the operator's key — do NOT hardcode a path)

```bash
# Pick an existing key, or create one. NEVER assume a specific filename.
KEY=$(ls ~/.ssh/id_ed25519 ~/.ssh/id_rsa 2>/dev/null | head -1)
if [ -z "$KEY" ]; then ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519; KEY=~/.ssh/id_ed25519; fi
# If the key has a passphrase, load it once (macOS keychain so it persists):
ssh-add --apple-use-keychain "$KEY" 2>/dev/null || ssh-add "$KEY"
# Test access (DO Ubuntu's default user is root):
ssh $SSH_OPTS root@$IP 'echo ok; . /etc/os-release; echo $PRETTY_NAME; docker --version'
```

If that returns `Permission denied (publickey)`, the droplet doesn't have this
machine's public key. Print it and have the user paste it into the droplet's
**Web Console** (DO panel → Droplet → Web Console):
```bash
cat "$KEY.pub"   # → user runs on the box: mkdir -p ~/.ssh && echo '<pubkey>' >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys
```
*(If `ssh-add` reports the key is passphrase-protected and the user must type it,
ask them to run `ssh-add --apple-use-keychain <key>` themselves via the session's
`!` prefix so the passphrase stays with them.)*

---

## 1. Harden + toolchain (on the box)

```bash
# Security update (the Docker-marketplace image is flagged out-of-date):
ssh $SSH_OPTS root@$IP 'export DEBIAN_FRONTEND=noninteractive NEEDRESTART_MODE=a;
  apt-get update -qq && apt-get upgrade -y -o Dpkg::Options::="--force-confold";
  [ -f /var/run/reboot-required ] && reboot || echo no-reboot'
sleep 15  # if it rebooted
# Node 20 (apt ships v12 — too old for corepack), rsync, Tailscale:
ssh $SSH_OPTS root@$IP '
  export DEBIAN_FRONTEND=noninteractive
  apt-get install -y -qq rsync
  curl -fsSL https://deb.nodesource.com/setup_20.x | bash - >/dev/null 2>&1
  apt-get install -y nodejs
  curl -fsSL https://tailscale.com/install.sh | sh'
# Join the tailnet, capture the address:
ssh $SSH_OPTS root@$IP "tailscale up --authkey $TS_KEY --hostname ${AGENT}-vps"
TS_DNS=$(ssh $SSH_OPTS root@$IP 'tailscale status --json | python3 -c "import sys,json;print(json.load(sys.stdin)[\"Self\"][\"DNSName\"].rstrip(\".\"))"')
echo "tailnet: $TS_DNS"   # e.g. ava-vps.tailXXXX.ts.net  (the user pastes this in the app)
```

---

## 2. Ship the source + fix Open Design for Linux

Find the KBDE checkout (`REPO=/path/to/KBDE-KyberAgent-Enterprise`) and rsync it.
The OD runtime ships macOS-native, so re-ship its payload and recompile its one
native dep for Linux:
```bash
cd "$REPO"
rsync -az --delete --exclude .git --exclude node_modules --exclude dist \
  --exclude 'apps/desktop/release' --exclude .tmp --exclude '*.log' \
  -e "ssh $SSH_OPTS" ./ root@$IP:/root/KBDE-KyberAgent-Enterprise/
# OD payload needs dist/ + node_modules — ship them, then rebuild better-sqlite3:
rsync -az -e "ssh $SSH_OPTS" ./apps/desktop/resources/apps/daemon/ \
  root@$IP:/root/KBDE-KyberAgent-Enterprise/apps/desktop/resources/apps/daemon/
ssh $SSH_OPTS root@$IP '
  apt-get install -y -qq build-essential python3
  d=/root/KBDE-KyberAgent-Enterprise/apps/desktop/resources/apps/daemon
  bs="$(find "$d/node_modules/.pnpm" -maxdepth 1 -type d -name "better-sqlite3@*" | head -1)/node_modules/better-sqlite3"
  cd "$bs" && npx --yes node-gyp rebuild
  cd "$d" && node -e "const {createRequire}=require(\"node:module\");const r=createRequire(process.cwd()+\"/dist/cli.js\");r(\"better-sqlite3\");r.resolve(\"@open-design/contracts\");console.log(\"OD_OK\")"
  # NodeSource already provides node/npm; stop the installer reinstalling old apt nodejs:
  sed -i "s/ca-certificates nodejs npm python3/ca-certificates python3/" /root/KBDE-KyberAgent-Enterprise/deploy/remote-vps/scripts/install-host.sh'
```

---

## 3. Run the host-native installer

```bash
ssh $SSH_OPTS root@$IP "cd /root/KBDE-KyberAgent-Enterprise &&
  KYBERAGENT_ADVERTISED_HOST=$TS_DNS \
  KYBERAGENT_OD_DEPS_SOURCE=/root/KBDE-KyberAgent-Enterprise/apps/desktop/resources/apps/daemon \
  sudo -E ./deploy/remote-vps/install.sh"
# Capture the daemon bearer (the user pastes this in the app). The authoritative
# source is remote.env; pairing.json also carries it under localApp.remoteDaemon.
BEARER=$(ssh $SSH_OPTS root@$IP "grep '^KYBERAGENT_DAEMON_TOKEN=' /etc/kyberagent/remote.env | cut -d= -f2")
echo "bearer: ${BEARER:0:8}…"
ssh $SSH_OPTS root@$IP 'for s in kyberagent-daemon kyberagent-live-artifacts open-design-daemon; do echo "$s: $(systemctl is-active $s)"; done'
```
The installer now provisions the **per-agent loopback-tcp listener** (`KA_PORT+1`)
+ `HOST_BRAIN_URL` automatically — env containers' brain proxy works out of the
box. (No manual `remote.env` edit needed anymore.)

---

## 4. Create the user's agent (NOT orchestrator)

```bash
B=$BEARER
ssh $SSH_OPTS root@$IP "curl -s -X POST -H 'Authorization: Bearer $B' -H 'Content-Type: application/json' \
  -d '{\"name\":\"$AGENT\",\"description\":\"cloud agent\"}' http://127.0.0.1:8765/agents/create"
# confirm:
ssh $SSH_OPTS root@$IP "curl -s -H 'Authorization: Bearer $B' http://127.0.0.1:8765/agents | python3 -c 'import sys,json;print([a[\"name\"] for a in json.load(sys.stdin)])'"
```
The agent is created with **local SQLite memory** by default (lives on the VPS).
To make additional agents later, repeat with a different name.

---

## 5. Claude login (device flow, reliable via tmux)

Run ONE login — multiple parallel flows cause `400` PKCE/state mismatches.
```bash
ssh $SSH_OPTS root@$IP '
  apt-get install -y -qq tmux
  tmux kill-session -t calogin 2>/dev/null
  tmux new-session -d -s calogin "sudo -iu KyberAgent claude auth login"
  sleep 9
  tmux capture-pane -t calogin -p -J | grep -oE "https://claude\.com/cai/oauth/authorize[^ ]*" | tail -1'
# → give that URL to the user; they authorize and return a "<code>#<state>".
ssh $SSH_OPTS root@$IP 'tmux send-keys -t calogin "<code>#<state>" Enter; sleep 6; sudo -iu KyberAgent claude auth status'
# expect loggedIn:true ; then pick up creds:
ssh $SSH_OPTS root@$IP 'tmux kill-session -t calogin 2>/dev/null; systemctl restart kyberagent-daemon'
```

---

## 6. Firewall (tailnet + Docker only)

```bash
ssh $SSH_OPTS root@$IP '
  ufw allow 22/tcp; ufw allow in on tailscale0; ufw allow in on docker0
  ufw default deny incoming; ufw default allow outgoing; ufw --force enable'
# verify from this machine (needs Tailscale running here, same tailnet):
curl -s -m6 -H "Authorization: Bearer $BEARER" http://$TS_DNS:8765/health   # works
curl -s -m6 http://$IP:8765/health || echo "public blocked (good)"
```

---

## 7. Docker / GitHub work-environment support

```bash
B=$BEARER
# (a) daemon (runs as user KyberAgent) needs Docker group access:
ssh $SSH_OPTS root@$IP 'usermod -aG docker KyberAgent && systemctl restart kyberagent-daemon'
# (b) pull the published env image + tag it as the local default. Use the latest
# published tag (check ghcr.io/kybernesisai/kyberagent-env) — 0.1.8-alpha shown here:
ENV_IMG=ghcr.io/kybernesisai/kyberagent-env:0.1.8-alpha
ssh $SSH_OPTS root@$IP "docker pull $ENV_IMG && docker tag $ENV_IMG kyberagent-env:latest"
# preflight should now read ready:true:
ssh $SSH_OPTS root@$IP "curl -s -H 'Authorization: Bearer $B' http://127.0.0.1:8765/environments/preflight"
# (c) connect GitHub on the box (device flow):
ssh $SSH_OPTS root@$IP "curl -s -X POST -H 'Authorization: Bearer $B' http://127.0.0.1:8765/github/connect"
# → give user the user_code + https://github.com/login/device ; they authorize + pick repos.
# poll GET /github/connect/<state> until status:connected.
```

---

## 8. Eval — verify end-to-end, then end clean

Run these as a checklist; every line should pass. `B=$BEARER`, on the box use
`http://127.0.0.1:8765`, off-box use `http://$TS_DNS:8765`.

```bash
# 1) services + daemon healthy
ssh $SSH_OPTS root@$IP "curl -s -H 'Authorization: Bearer $B' http://127.0.0.1:8765/health"          # {"ok":true,...}
# 2) the agent exists and its (empty) brain reads
ssh $SSH_OPTS root@$IP "curl -s -H 'Authorization: Bearer $B' http://127.0.0.1:8765/brain/$AGENT/stats"  # {"timeline_rows":0,...}
# 3) model auth is live (THIS is what lets the agent think)
ssh $SSH_OPTS root@$IP 'sudo -iu KyberAgent claude auth status'                                       # loggedIn:true, subscriptionType
# 4) a Dockerized env provisions, clones, and proxies brain back (only if GitHub connected):
EID=$(ssh $SSH_OPTS root@$IP "curl -s -X POST -H 'Authorization: Bearer $B' -H 'Content-Type: application/json' -d '{\"repo\":\"<owner>/<small-repo>\",\"agent\":\"$AGENT\"}' http://127.0.0.1:8765/environments | python3 -c 'import sys,json;print(json.load(sys.stdin)[\"id\"])'")
#    poll /environments/$EID → status:ready ; container: /workspace/repo cloned on branch kyberagent/$AGENT-...
#    env brain (busToken → :KA_PORT+1) returns the agent's stats → host-proxy works.
# 5) TEAR DOWN the eval env so nothing is left running:
ssh $SSH_OPTS root@$IP "curl -s -X DELETE -H 'Authorization: Bearer $B' http://127.0.0.1:8765/environments/$EID >/dev/null && echo torn-down"
```
A full *model turn* is best confirmed by the user sending a message from the
desktop after pairing (Step 9) — that exercises desktop→VPS→agent→Claude live. If
you have a turn-dispatch path available, dispatch "say hello" and poll the
transcript for an assistant reply; otherwise hand off to the user for that final
click.

Cleanup: remove any test env (step 5 above), kill leftover tmux sessions, and
revoke the Tailscale auth key if it was single-use.

---

## 9. Output to the user (the three pairing values)

Print these for the desktop app → AGENTS → **⊕ ADD REMOTE → MANUAL PEER**:

| Field | Value |
|---|---|
| Peer agent name | `$AGENT` |
| Peer URL | `$TS_DNS` |
| Port | `8765` |
| Bearer token | `$BEARER` |

(Open Design + Live Artifacts URLs/bearers from `/etc/kyberagent/pairing.json`
go in the app's Design / Live Artifacts runtime settings — optional.)

---

## Gotchas (each cost real debugging the first time)

1. **SSH key passphrase** → `ssh-add --apple-use-keychain <key>` once; otherwise non-interactive SSH fails with `Permission denied (publickey)` even though the server *accepts* the key.
2. **apt Node is v12** → install NodeSource 20 + drop `nodejs npm` from the installer's apt line.
3. **Open Design is macOS-native** → re-ship its payload (dist + node_modules) and recompile `better-sqlite3` for Linux; set `KYBERAGENT_OD_DEPS_SOURCE`.
4. **Daemon can't see Docker** → add `KyberAgent` to the `docker` group + restart.
5. **Env image absent** → it's never built on the box; **pull** the published GHCR image and tag `kyberagent-env:latest`.
6. **Claude login 400** → only ever run ONE login flow; use tmux + `send-keys`.
7. **Create the agent** → don't reuse `orchestrator`; create a named agent (`/agents/create`) and pair THAT.
8. **Manual Peer** uses the **agent name** as "Peer agent name"; the agent must exist on the box first.

## Result
An always-on cloud agent: its own SQLite brain on the VPS, signed into the user's
Claude plan, paired to the desktop over Tailscale, hardened (tailnet + Docker
only), able to provision isolated Dockerized GitHub work-environments — with the
desktop's per-agent proxy routing env/GitHub/filesystem to the VPS.
