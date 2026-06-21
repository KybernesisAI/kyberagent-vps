# Connecting to a KyberAgent VPS

Every operation in this plugin reaches a deployed KyberAgent box two ways: **SSH
as root** (system ops) and the **daemon's bearer-gated HTTP API** (agent ops).
Each skill includes the minimal bootstrap it needs; this file is the full
reference for the connection + the fixed install layout.

## Bootstrap

```bash
VPS=<ip-or-tailnet-host>        # e.g. 159.223.80.165  OR  kyberagent-vps.tailXXXX.ts.net
# A function, NOT a string var — `$SSH "…"` breaks under zsh (no word-splitting).
kssh() { ssh -o ConnectTimeout=20 -o StrictHostKeyChecking=accept-new root@"$VPS" "$@"; }
kssh true || echo "SSH failed — resolve a key below"
```

SSH key — resolve the operator's key, never hardcode a path:
```bash
KEY=$(ls ~/.ssh/id_ed25519 ~/.ssh/id_rsa 2>/dev/null | head -1)
ssh-add --apple-use-keychain "$KEY" 2>/dev/null || ssh-add "$KEY"
```

Daemon token + advertised peer URL (source of truth for every API call + desktop
pairing) and an `api()` helper that calls the bearer-gated daemon over loopback:
```bash
BEARER=$(kssh "grep '^KYBERAGENT_DAEMON_TOKEN=' /etc/kyberagent/remote.env | cut -d= -f2-")
PEER=$(kssh "grep '^KYBERAGENT_DAEMON_ADVERTISED_ENDPOINT=' /etc/kyberagent/remote.env | cut -d= -f2-")
api() { kssh "curl -s -m12 -H 'Authorization: Bearer $BEARER' $*"; }
# usage: api http://127.0.0.1:8765/agents
```

## Install layout (fixed — don't guess)

| Thing | Value |
|---|---|
| App source | `/opt/kyberagent/app` |
| State / agent homes | `/var/lib/kyberagent/.kyberagent` (agents under `…/agents/<name>`) |
| Env file | `/etc/kyberagent/remote.env` |
| App user | `KyberAgent` |
| Daemon (network-tcp, bearer-gated) | port **8765** — `kyberagent-daemon.service` |
| Env-brain (loopback-tcp, for containers) | port **8766** (= daemon + 1) |
| Live Artifacts | port **8788** — `kyberagent-live-artifacts.service` |
| Open Design | port **7456** — `open-design-daemon.service` |
| CLI shim | `/usr/local/bin/kyberagent` (run as the `KyberAgent` user) |

## Conventions
- API calls go to `127.0.0.1:8765` **via SSH** so they hit the bearer-gated
  network listener locally. You can also call `$PEER` directly over the tailnet
  with the same bearer if your machine is on the tailnet.
- The bearer in `remote.env` is sensitive — fine to hand the user for desktop
  pairing, but don't paste it into shared logs or PRs.
- Run the `kyberagent` CLI as the app user: `kssh "sudo -iu KyberAgent kyberagent …"`.
