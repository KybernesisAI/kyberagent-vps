---
name: connect-arcana
description: Move a VPS agent's brain from local SQLite to Arcana cloud memory using the headless RFC 8628 device flow (no browser or tunnel needed on the server), or disconnect it back to local. Use when the user asks to "connect <agent> to Arcana", "put the agent's memory in the cloud", "switch <agent> to the cloud brain", or "disconnect <agent> from Arcana".
allowed-tools: Bash
---

# Connect a VPS agent's brain to Arcana (device flow)

A VPS agent has no local browser, so it binds Arcana via the **device flow**: the
daemon gets a code, the user approves it in any browser, the daemon polls + binds.
See `../../reference/connection.md`.

## 1. Connect
```bash
VPS=<ip-or-tailnet-host>
AGENT=<name>
kssh() { ssh -o ConnectTimeout=20 -o StrictHostKeyChecking=accept-new root@"$VPS" "$@"; }
BEARER=$(kssh "grep '^KYBERAGENT_DAEMON_TOKEN=' /etc/kyberagent/remote.env | cut -d= -f2-")
api() { kssh "curl -s -m12 -H 'Authorization: Bearer $BEARER' $*"; }
```

## 2. Start the device flow
```bash
kssh "curl -s -X POST -H 'Authorization: Bearer $BEARER' -H 'Content-Type: application/json' \
  -d '{\"arcana_url\":\"https://api.arcana.kybernesis.ai\",\"mode\":\"device\"}' \
  http://127.0.0.1:8765/agents/$AGENT/cloud/connect"
# → { state, user_code, verification_uri, verification_uri_complete, interval, expires_in }
```

## 3. Hand the user the code
Give them `verification_uri_complete` (code pre-filled) or the `user_code` +
`verification_uri`. They open it, sign in, **pick the workspace**, approve. The
daemon polls Arcana in the background and binds on approval (~15 min window).

## 4. Verify
```bash
api http://127.0.0.1:8765/agents/$AGENT/cloud/status     # { connected:true, workspace:… }
api http://127.0.0.1:8765/brain/$AGENT/stats             # vector_backend: arcana-remote
```

## Disconnect (revert to local SQLite on the box)
```bash
kssh "curl -s -X POST -H 'Authorization: Bearer $BEARER' http://127.0.0.1:8765/agents/$AGENT/cloud/disconnect"
```
The local SQLite brain (a pre-migration snapshot) is untouched and becomes active again.

> From the desktop app this is one-click: Settings → the agent → ARCANA CLOUD
> BRAIN → Connect shows the code. This skill is the headless/scripted equivalent.
