---
name: health-check
description: Run a full health sweep of a KyberAgent VPS — systemd services, daemon API reachability, the agent roster, Docker + the environment image, and disk/memory. Use when the user asks "check the VPS", "is the box healthy", "is everything running", "is the daemon up", or wants a status report of a deployed KyberAgent server.
allowed-tools: Bash
---

# Health check a KyberAgent VPS

Report one green/red line per check. See `../../reference/connection.md` for the
full layout. Ask the user for the VPS host (IP or tailnet name) if you don't have it.

## 1. Connect
```bash
VPS=<ip-or-tailnet-host>
kssh() { ssh -o ConnectTimeout=20 -o StrictHostKeyChecking=accept-new root@"$VPS" "$@"; }
BEARER=$(kssh "grep '^KYBERAGENT_DAEMON_TOKEN=' /etc/kyberagent/remote.env | cut -d= -f2-")
api() { kssh "curl -s -m12 -H 'Authorization: Bearer $BEARER' $*"; }
```

## 2. Sweep
```bash
echo "── services ──"
kssh "systemctl is-active kyberagent-daemon kyberagent-live-artifacts open-design-daemon"
echo "── daemon API ──"
api http://127.0.0.1:8765/health
echo "── agents (name · kind · backend) ──"
api http://127.0.0.1:8765/agents | python3 -c 'import sys,json
for a in json.load(sys.stdin): print(" ", a["name"], a.get("kind","local"), a.get("memory_backend") or "local")'
echo "── docker + env image ──"
api http://127.0.0.1:8765/environments/preflight
echo "── disk + mem ──"
kssh "df -h / | tail -1; free -h | grep Mem"
```

## 3. Interpret + report
- All three services `active` → daemon stack healthy.
- `/health` returns `{"ok":true,…}` → API reachable.
- preflight `dockerRunning:true, imagePresent:true, ready:true` → envs will work.
- Red flags: a service `inactive`/`failed` → use the **update-build** or restart
  it (`kssh "systemctl restart <svc>"`) and tail `kssh "journalctl -u <svc> -n 80 --no-pager"`;
  `imagePresent:false` → use the **check-docker** skill; `/health` unreachable but
  service active → re-check the bearer/port.

Summarize as a short status table for the user.
