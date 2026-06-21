---
name: connect-desktop
description: Fetch the values needed to connect a VPS agent to the KyberAgent desktop app — the bearer token, the advertised peer URL, the port, and the agent name, formatted for the desktop's "Add Remote → Manual Peer" form. Use when the user asks "how do I add my agent to the desktop", "get the bearer token", "what's the peer URL/token to pair", or "give me the connection details for <agent>".
allowed-tools: Bash
---

# Get desktop-pairing values for a VPS agent

The exact fields the user pastes into the desktop's **⊕ ADD REMOTE → MANUAL PEER**
form. See `../../reference/connection.md`.

## 1. Connect
```bash
VPS="$(cat ~/.config/kyberagent-vps/host 2>/dev/null)"   # saved on first use
# If $VPS is empty: ask the user for their KyberAgent box's host — its Tailscale
# name or public IP. They got it from the `deploy` skill's final output, or it's
# the droplet IP in their cloud dashboard / the machine in their Tailscale admin.
# Then save it so no skill asks again:
#   mkdir -p ~/.config/kyberagent-vps && printf %s "<host>" > ~/.config/kyberagent-vps/host
# (If kssh later fails with "Permission denied", their SSH key isn't authorized on
# the box as root — root SSH access is required; the deploy skill sets this up.)
kssh() { ssh -o ConnectTimeout=20 -o StrictHostKeyChecking=accept-new root@"$VPS" "$@"; }
BEARER=$(kssh "grep '^KYBERAGENT_DAEMON_TOKEN=' /etc/kyberagent/remote.env | cut -d= -f2-")
PEER=$(kssh "grep '^KYBERAGENT_DAEMON_ADVERTISED_ENDPOINT=' /etc/kyberagent/remote.env | cut -d= -f2-")
```

## 2. Confirm the agent name exists (it must match a real agent)
```bash
kssh "curl -s -H 'Authorization: Bearer $BEARER' http://127.0.0.1:8765/agents | python3 -c 'import sys,json;print([a[\"name\"] for a in json.load(sys.stdin)])'"
```

## 3. Present the pairing values
```bash
echo "Peer agent name : <agent-name>     # must match one from step 2, e.g. ava"
echo "Peer URL        : $PEER            # the host part; the form has a separate Port field"
echo "Port            : 8765"
echo "Bearer token    : $BEARER"
echo "Shared sessions : off"
```
`$PEER` looks like `http://kyberagent-vps.tailXXXX.ts.net:8765` — host =
`kyberagent-vps.tailXXXX.ts.net`, port = `8765`. The bearer is the **same daemon
token for every agent** on the box (it gates the daemon, not the agent). The
user's machine must be on the same **Tailscale** network to reach the peer URL.
