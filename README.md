# KyberAgent VPS — Claude Code plugin

A Claude Code **plugin marketplace** containing the **`kyberagent-vps`** plugin:
deploy and operate a [KyberAgent](https://github.com/KybernesisAI) cloud agent on
a VPS, with **one skill per task** so Claude Code can manage the box on command.

## Install

```text
/plugin marketplace add KybernesisAI/kyberagent-vps
/plugin install kyberagent-vps@kybernesis
```

Then just ask, e.g. *"check the VPS health"*, *"create an agent named atlas on the
VPS"*, *"update the VPS to the new build"*, *"get the token to add ava to the
desktop"*. Claude picks the matching skill automatically; you can also invoke one
directly as `/kyberagent-vps:<skill>`.

## Skills

| Skill | What it does |
|---|---|
| `deploy` | First-time, end-to-end install of a fresh VPS (droplet → hardening → Tailscale → daemon/CLI/Live Artifacts/Open Design → your agent → Docker envs → eval). |
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
        ├── reference/connection.md   # shared SSH + layout reference
        └── skills/<task>/SKILL.md     # one skill per task
```

## Requirements

- A KyberAgent VPS deployed by the `deploy` skill (or the manual installer).
- SSH access to the box (as `root`) and the daemon bearer token in
  `/etc/kyberagent/remote.env` — every skill resolves these for you.
- Tailscale on your machine to reach the agent from the desktop app.

## License

MIT © KybernesisAI
