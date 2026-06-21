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
VPS=<ip-or-tailnet-host>
kssh() { ssh -o ConnectTimeout=20 -o StrictHostKeyChecking=accept-new root@"$VPS" "$@"; }
BEARER=$(kssh "grep '^KYBERAGENT_DAEMON_TOKEN=' /etc/kyberagent/remote.env | cut -d= -f2-")
api() { kssh "curl -s -m12 -H 'Authorization: Bearer $BEARER' $*"; }
```

## 2. List
```bash
api http://127.0.0.1:8765/agents | python3 -c 'import sys,json
for a in json.load(sys.stdin):
    print(f"{a[\"name\"]:14} kind={a.get(\"kind\",\"local\")} backend={a.get(\"memory_backend\") or \"local\"} workspace={a.get(\"cloud_workspace\")}")'
```
Or via the CLI: `kssh "sudo -iu KyberAgent kyberagent agents"`.

`orchestrator` is the built-in agent; named agents (e.g. `ava`) are the ones the
user created. To add one use the **create-agent** skill; for the values to pair
one to the desktop use **connect-desktop**.
