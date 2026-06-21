# Connecting to a KyberAgent VPS

Every operation in this plugin reaches a deployed KyberAgent box two ways: **SSH
as root** (system ops) and the **daemon's bearer-gated HTTP API** (agent ops).

## Prerequisites (what a user needs before any ops skill works)

1. **A deployed KyberAgent VPS.** If you don't have one, run the **`deploy`**
   skill first (`/kyberagent-vps:deploy`) — it provisions the server and prints
   your host, agent name, and pairing values at the end.
2. **Root SSH access to the box.** Your SSH key must be authorized as `root` on
   the server. The `deploy` skill sets this up; if you got the box another way,
   add your public key to the server first. (A `Permission denied` from `kssh`
   means your key isn't authorized.)
3. **The host** — the server's **Tailscale name** (e.g.
   `kyberagent-vps.tailXXXX.ts.net`) or its **public IP** (e.g. `159.223.80.165`).
   Where to find it: the `deploy` skill's final output; or the droplet's IP in
   your cloud dashboard (DigitalOcean/Hetzner); or the machine in your
   [Tailscale admin](https://login.tailscale.com/admin/machines).

You do **not** need to find the bearer token yourself — every skill reads it from
the box over SSH.

## Resolving the host (asked once, then saved)

Skills load the host from `~/.config/kyberagent-vps/host`. On first use, ask the
user (per the prerequisites above) and save it so nothing asks again:

```bash
VPS="$(cat ~/.config/kyberagent-vps/host 2>/dev/null)"
# if empty: get it from the user, then —
mkdir -p ~/.config/kyberagent-vps && printf %s "<host>" > ~/.config/kyberagent-vps/host
```

## Bootstrap

```bash
# A function, NOT a string var — `$SSH "…"` breaks under zsh (no word-splitting).
kssh() { ssh -o ConnectTimeout=20 -o StrictHostKeyChecking=accept-new root@"$VPS" "$@"; }
kssh true || echo "SSH failed — your key isn't authorized as root on the box"

# Daemon token + advertised peer URL (source of truth for API + desktop pairing),
# and an api() helper that calls the bearer-gated daemon over loopback:
BEARER=$(kssh "grep '^KYBERAGENT_DAEMON_TOKEN=' /etc/kyberagent/remote.env | cut -d= -f2-")
PEER=$(kssh "grep '^KYBERAGENT_DAEMON_ADVERTISED_ENDPOINT=' /etc/kyberagent/remote.env | cut -d= -f2-")
api() { kssh "curl -s -m12 -H 'Authorization: Bearer $BEARER' $*"; }
# usage: api http://127.0.0.1:8765/agents
```

SSH key not loaded? Resolve the operator's key (never hardcode a path):
```bash
KEY=$(ls ~/.ssh/id_ed25519 ~/.ssh/id_rsa 2>/dev/null | head -1)
ssh-add --apple-use-keychain "$KEY" 2>/dev/null || ssh-add "$KEY"
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
- Python parsing: a `python3 -c` **after a local pipe** is single-quoted — use
  plain `"` and `%`-formatting (no f-strings with nested quotes). A `python3 -c`
  run **on the box inside `kssh "…"`** must escape inner quotes as `\"`.
