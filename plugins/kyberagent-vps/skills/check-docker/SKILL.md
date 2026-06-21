---
name: check-docker
description: Verify and fix the Docker + environment-image chain on a KyberAgent VPS so remote work-environments (Dockerized GitHub/VPS sessions) provision correctly. Checks the Docker daemon, the app user's docker-group membership, presence of the kyberagent-env image (pulls if missing), and the env-brain loopback wiring. Use when the user asks "is Docker OK on the box", "does the VPS have the env image", "remote environments won't spin up", or "is the Docker environment connected correctly".
allowed-tools: Bash
---

# Verify Docker + the environment image on a KyberAgent VPS

Remote sessions spin up Docker containers from the env image, and those
containers reach the host daemon's brain over a loopback port. Verify the whole
chain. See `../../reference/connection.md`.

## 1. Connect
```bash
VPS=<ip-or-tailnet-host>
kssh() { ssh -o ConnectTimeout=20 -o StrictHostKeyChecking=accept-new root@"$VPS" "$@"; }
BEARER=$(kssh "grep '^KYBERAGENT_DAEMON_TOKEN=' /etc/kyberagent/remote.env | cut -d= -f2-")
api() { kssh "curl -s -m12 -H 'Authorization: Bearer $BEARER' $*"; }
```

## 2. Check
```bash
echo "── daemon's view (authoritative) ──"
api http://127.0.0.1:8765/environments/preflight
#   want: dockerInstalled:true, dockerRunning:true, imagePresent:true, ready:true
echo "── docker daemon + app user in docker group ──"
kssh "systemctl is-active docker; id KyberAgent | grep -o docker || echo 'KyberAgent NOT in docker group'"
echo "── env image tags present ──"
kssh "docker images --format '{{.Repository}}:{{.Tag}}' | grep -i kyberagent-env || echo 'MISSING env image'"
echo "── env-brain wiring (containers reach the host brain here) ──"
kssh "grep -E 'KYBERAGENT_LEGACY_WEB_PORT|KYBERAGENT_HOST_BRAIN_URL|KYBERAGENT_TCP_HOST' /etc/kyberagent/remote.env"
```

## 3. Fix
- `KyberAgent NOT in docker group` → `kssh "usermod -aG docker KyberAgent && systemctl restart kyberagent-daemon"`.
- `MISSING env image` / `imagePresent:false` → `kssh "docker pull ghcr.io/kybernesisai/kyberagent-env:latest"` (the daemon also auto-pulls on first provision; pre-pulling avoids a slow first session).
- containers can't reach the host brain (env-brain 401/refused) → confirm
  `KYBERAGENT_TCP_HOST=0.0.0.0`, `KYBERAGENT_HOST_BRAIN_URL=http://host.docker.internal:8766`,
  and that the daemon listens on the loopback-tcp port (8766).
- stale env containers piling up → `kssh "docker ps -a --filter name=kyberagent --format '{{.ID}} {{.Status}}'"`; the reaper handles these, `docker rm -f <id>` clears stragglers.
