---
name: create-agent
description: Create a new named agent on a KyberAgent VPS and prepare it for use — scaffold its identity + bundled skills, verify registration, hand back the desktop-pairing values, and optionally connect its brain to Arcana. Use when the user asks to "create an agent on the VPS", "add a new agent named X", "spin up a new cloud agent", or "make me an agent on the box".
allowed-tools: Bash
---

# Create a new agent on a KyberAgent VPS

Creates a named agent (its own SOUL/USER/CLAUDE/identity + bundled skills) on the
box, then sets it up for the desktop. See `../../reference/connection.md`.

## 1. Collect + connect
Ask the user for: the **agent name** (single lowercase word, `^[a-z][a-z0-9-]{1,30}$`)
and a **one-line role/description**. Optionally a persona and who they work for.
```bash
VPS=<ip-or-tailnet-host>
NAME=<lowercase-name>
kssh() { ssh -o ConnectTimeout=20 -o StrictHostKeyChecking=accept-new root@"$VPS" "$@"; }
BEARER=$(kssh "grep '^KYBERAGENT_DAEMON_TOKEN=' /etc/kyberagent/remote.env | cut -d= -f2-")
PEER=$(kssh "grep '^KYBERAGENT_DAEMON_ADVERTISED_ENDPOINT=' /etc/kyberagent/remote.env | cut -d= -f2-")
api() { kssh "curl -s -m12 -H 'Authorization: Bearer $BEARER' $*"; }
```

## 2. Create (CLI — scaffolds the home correctly)
```bash
kssh "sudo -iu KyberAgent kyberagent agent create $NAME --description 'one-line role'"
#   optional: --persona '<voice/style>'  --user-brief '<who they work for>'  --package <bundled-agent-id>
```
Fallback if the CLI subcommand differs on this build (daemon API):
```bash
kssh "curl -s -X POST -H 'Authorization: Bearer $BEARER' -H 'Content-Type: application/json' -d '{\"name\":\"$NAME\",\"description\":\"one-line role\"}' http://127.0.0.1:8765/agents/create"
```

## 3. Verify
```bash
api http://127.0.0.1:8765/agents | python3 -c 'import sys,json;print([a["name"] for a in json.load(sys.stdin)])'
```
Confirm `$NAME` is listed.

## 4. Hand back the desktop-pairing values
The new agent uses the same daemon bearer. Give the user the MANUAL PEER fields:
```bash
echo "Peer agent name : $NAME"
echo "Peer URL        : $PEER       (host part; Port field = 8765)"
echo "Port            : 8765"
echo "Bearer token    : $BEARER"
echo "Shared sessions : off"
```
(See the **connect-desktop** skill for the same values for any existing agent.)

## 5. Brain (optional)
New agents start on the **local SQLite** brain. To move the agent to Arcana
cloud, use the **connect-arcana** skill. Done — report the name + pairing values.
