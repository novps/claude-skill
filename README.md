# novps-claude-skill

A [Claude Code](https://claude.com/claude-code) skill for deploying and operating applications on [novps.io](https://novps.io) without leaving the terminal. Say *"deploy this to novps"* or *"tail the prod logs"* and the agent runs the right `novps` CLI commands.

## What it does

- **Deploy** via manifests: `novps apps apply <name> -f novps.yaml --wait`
- **Ship** a new image: `novps resources set-image` + `resources deploy`
- **Observe**: tail logs, shell into pods, port-forward databases
- **Manage**: scale, edit envs, rollback, redeploy
- **Bootstrap**: generates a `novps.yaml` from an existing app via `novps apps export`

Supports both deploy modes:
- **Docker** — pull a pre-built image from any registry (ghcr, novps registry, Docker Hub)
- **GitHub** — novps builds from source on each deploy (requires novps GitHub App installed)

## Requirements

- `novps` CLI **v0.3.0 or later** — install from [cli.novps.io](https://cli.novps.io)
- A novps Personal Access Token — `novps auth login --token nvps_...`
- Claude Code (or any Claude Agent SDK harness that loads skills)

## Installation

The install directory must be named `novps` — Claude Code uses the directory name as the skill name.

### As a project skill (this repo)

Clone or download this repo into your project's `.claude/skills/novps/` directory:

```bash
mkdir -p .claude/skills
git clone https://github.com/novps/claude-skill .claude/skills/novps
```

### As a user-level skill (all projects)

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/novps/claude-skill ~/.claude/skills/novps
```

Claude Code will auto-load the skill on matching intents.

## Usage

Once installed, just talk to your agent:

- *"Deploy this app to novps"*
- *"Bump the image tag to the current commit on the api resource"*
- *"Tail the prod logs for the last 15 minutes and grep for errors"*
- *"Give me a shell into the worker pod"*
- *"Port-forward the prod database to localhost"*
- *"Scale the api to sm:4"*

The skill will resolve resource IDs, pick the right CLI flow, surface confirmations for destructive actions, and never commit secrets to the manifest.

## Repository layout

```
.
├── SKILL.md                       # entrypoint loaded by Claude Code
├── README.md                      # this file
├── reference/
│   ├── manifest-schema.md         # full novps.yaml schema
│   ├── novps.docker.example.yaml  # pre-built image starter
│   └── novps.github.example.yaml  # build-from-source starter
└── LICENSE
```

## License

MIT — see [LICENSE](./LICENSE).
