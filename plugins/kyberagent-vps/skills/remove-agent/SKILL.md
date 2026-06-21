---
name: remove-agent
description: Remove an agent from a KyberAgent VPS — disconnect its cloud brain if connected, unregister it from the daemon, and optionally delete its home directory. Use when the user asks to "remove <agent> from the VPS", "delete the agent on the box", "unregister <agent>", or "tear down a test agent". Never removes the built-in orchestrator.
allowed-tools: Bash
---

# Remove an agent from a KyberAgent VPS

See `../../reference/connection.md`. **Confirm with the user before deleting a
home dir** — it's irreversible (local brain + identity). Never remove `orchestrator`.

## 1. Connect
```bash
VPS="$(cat ~/.config/kyberagent-vps/host 2>/dev/null)"
# ↑ The saved host. If it is NON-EMPTY, USE IT and continue — do NOT ask the user.
# ONLY if it is empty: ask for the box's Tailscale name or IP (from the `deploy`
# output, the cloud dashboard, or the Tailscale admin), then save it so nothing
# asks again:  mkdir -p ~/.config/kyberagent-vps && printf %s "<host>" > ~/.config/kyberagent-vps/host
# ("Permission denied" from kssh = your SSH key isn't root-authorized on the box.)
AGENT=<name>
kssh() { ssh -o ConnectTimeout=20 -o StrictHostKeyChecking=accept-new root@"$VPS" "$@"; }
BEARER=$(kssh "grep '^KYBERAGENT_DAEMON_TOKEN=' /etc/kyberagent/remote.env | cut -d= -f2-")
```
Refuse if `$AGENT` is `orchestrator`.

## 2. Disconnect cloud first (if cloud-backed — drains the outbound queue)
```bash
kssh "curl -s -X POST -H 'Authorization: Bearer $BEARER' http://127.0.0.1:8765/agents/$AGENT/cloud/disconnect"
```

## 3. Unregister (CLI; does NOT delete the home dir)
```bash
kssh "sudo -iu KyberAgent kyberagent agent remove $AGENT"
```

## 4. (Optional, destructive) delete the home dir
Only after explicit user confirmation:
```bash
kssh "rm -rf /var/lib/kyberagent/.kyberagent/agents/$AGENT"
```

## 5. Verify
```bash
kssh "curl -s -H 'Authorization: Bearer $BEARER' http://127.0.0.1:8765/agents | python3 -c 'import sys,json;print([a[\"name\"] for a in json.load(sys.stdin)])'"
```
Confirm `$AGENT` is gone and `orchestrator` (+ any others) remain.
