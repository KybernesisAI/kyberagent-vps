---
name: list-agents
description: List the agents registered on a KyberAgent VPS, with each agent's kind (local/remote), memory backend (local SQLite vs Arcana cloud), and bound workspace. Use when the user asks "list the agents on the VPS", "what agents are on the box", "which agents exist", or "is <agent> on the server".
allowed-tools: Bash
---

# List agents on a KyberAgent VPS

See `../../reference/connection.md` for the full layout. Ask for the VPS host if
you don't have it.

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
api() { kssh "curl -s -m12 -H 'Authorization: Bearer $BEARER' $*"; }
```

## 2. List
```bash
api http://127.0.0.1:8765/agents | python3 -c 'import sys,json
rows=json.load(sys.stdin)
print("%-14s %-8s %-7s %s" % ("AGENT","KIND","BACKEND","WORKSPACE"))
for a in rows:
    print("%-14s %-8s %-7s %s" % (a["name"], a.get("kind","local"), a.get("memory_backend") or "local", a.get("cloud_workspace") or "-"))'
```
Or via the CLI: `kssh "sudo -iu KyberAgent kyberagent agents"`.

> Note: the `python3 -c` runs **locally** (after the pipe), so it's inside a
> single-quoted block — use plain `"` for dict keys and avoid f-strings with
> nested quotes (use `%`-formatting). When a `python3 -c` runs **on the box**
> inside `kssh "…"`, the inner quotes must be escaped (`\"`) to survive the outer
> double-quote — see the create-agent / remove-agent skills.

`orchestrator` is the built-in agent; named agents (e.g. `ava`) are the ones the
user created. To add one use the **create-agent** skill; for the values to pair
one to the desktop use **connect-desktop**.
