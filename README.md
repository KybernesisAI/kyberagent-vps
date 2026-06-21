# KyberAgent VPS — Claude Code plugin

A Claude Code **plugin marketplace** containing the **`kyberagent-vps`** plugin:
deploy and operate a [KyberAgent](https://github.com/KybernesisAI) cloud agent on
a VPS, with **one skill per task** so Claude Code can manage the box on command.

## Install

```text
/plugin marketplace add KybernesisAI/kyberagent-vps
/plugin install kyberagent-vps@kybernesis
```

## Getting started

### If you don't have a server yet
Run **`/kyberagent-vps:deploy`**. It stands up a fresh VPS end-to-end and, at the
end, prints your **host**, your **agent name**, and the **desktop-pairing values**.
You'll need:
- a **DigitalOcean** or **Hetzner** account (the server, ~$48/mo),
- a **Tailscale** account + an auth key (free; private networking),
- a **Claude Pro/Max** subscription (the agent's thinking).

### If you already have a KyberAgent VPS
Just ask, e.g. *"check the VPS health"*, *"create an agent named atlas on the VPS"*,
*"update the VPS to the new build"*, *"get the token to add ava to the desktop"*.
The first skill you run asks for your **host** once and saves it to
`~/.config/kyberagent-vps/host`, so nothing asks again.

**What each skill needs from you:**
- **Host** — your server's Tailscale name (`kyberagent-vps.tailXXXX.ts.net`) or
  public IP. Get it from the `deploy` output, your cloud dashboard, or your
  [Tailscale admin](https://login.tailscale.com/admin/machines).
- **Root SSH access** — your SSH key must be authorized as `root` on the box
  (the `deploy` skill sets this up). A `Permission denied` means it isn't.
- That's it — the daemon token and all other connection details are read from the
  box automatically.

You can also invoke a skill directly as `/kyberagent-vps:<skill>`.

## Skills

| Skill | What it does |
|---|---|
| `deploy` | First-time, end-to-end install of a fresh VPS (droplet → hardening → Tailscale → daemon/CLI/Live Artifacts/Open Design → your agent → Docker envs → eval). Prints your host + pairing values. |
| `health-check` | Full health sweep — services, daemon API, agent roster, Docker + env image, disk/mem. |
| `create-agent` | Create a named agent, verify it, hand back desktop-pairing values, optionally connect Arcana. |
| `list-agents` | List agents with kind / memory backend / workspace. |
| `connect-desktop` | Fetch the bearer token + peer URL + port to pair an agent into the desktop app. |
| `update-build` | Deploy a new release — sync source, rebuild the right artifact (daemon bundle / CLI / Live Artifacts), restart. |
| `check-docker` | Verify + fix the Docker + environment-image chain for remote work-environments. |
| `connect-arcana` | Move an agent's brain to Arcana cloud via the headless device flow (or disconnect). |
| `remove-agent` | Disconnect cloud, unregister, optionally delete the home (never `orchestrator`). |

## Layout

```text
.
├── .claude-plugin/
│   └── marketplace.json          # marketplace catalog (name: kybernesis)
└── plugins/
    └── kyberagent-vps/
        ├── .claude-plugin/plugin.json
        ├── reference/connection.md   # shared SSH + host-resolution + layout reference
        └── skills/<task>/SKILL.md     # one skill per task
```

## License

MIT © KybernesisAI
