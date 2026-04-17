---
name: novps
description: Deploy applications to novps.io and run observability/infra tasks against novps resources. Use when the user mentions novps, novps.io, asks to deploy, ship, or release to novps, wants novps logs, a shell into a novps pod, to port-forward to a novps database, or to edit a novps.yaml manifest. Also use when a novps.yaml is present in the repo.
---

# novps

Native deploy and observability for [novps.io](https://novps.io) via the `novps` CLI (v0.3.0+). Wraps the CLI in a set of clear flows so an agent can take intents like "deploy to novps", "tail prod logs", or "bump the image" and turn them into the right command chain.

## When to use this skill

Trigger on any of:
- The user says *novps*, *novps.io*, "deploy to novps", "ship to novps", "push to novps".
- The user asks for logs, a shell, a port-forward, or secrets tied to novps.
- A `novps.yaml` exists in the repo.
- The user wants to scale, rollback, or restart a novps resource.

## Prereqs — always check first

Run these before any action. If any fail, stop and surface the exact fix.

```bash
novps version          # must be >= 0.3.0
novps auth status      # must show authenticated
```

- CLI missing → install from `cli.novps.io`.
- Not authenticated → `novps auth login` (the CLI will prompt for the token interactively; prefer this over passing `--token` in a shared terminal, since the token lands in shell history). The user must generate a PAT (prefix `nvps_`) in the novps dashboard first.
- Wrong project → every command accepts `--project/-p <alias>`; default alias is `default`.

## The deploy flow

Novps is **manifest-driven**. The source of truth is `novps.yaml` at the repo root. The app name is not in the YAML — it's the CLI argument.

### 1. Has a `novps.yaml`?

```bash
novps apps apply <app-name> -f novps.yaml --wait
```

- `apps apply` **creates the app if it doesn't exist** and updates it if it does. No separate "create" step needed.
- Resolve the manifest path from the repo root, not cwd — if the agent is invoked from a subdirectory, `-f` must still point at the real location.
- Add `--env-file .env` if a `.env` exists in the repo, so `${VAR}` placeholders resolve. `${VAR}` is filled from the merged shell env + `.env` file.
- Add `--dry-run` first any time `--prune` is being used or when the manifest was edited significantly. Show the user the dry-run output, get confirmation, then re-run without `--dry-run`.
- `--wait` blocks until rollout finishes. Pair with `--json` when the agent needs to parse the result rather than show it to the user.

### 2. No manifest yet? Bootstrap.

```bash
novps apps list                         # pick the app
novps apps export <app-name> -o novps.yaml
```

Do **not** pass `--include-secrets` by default. If the user explicitly wants secret values in the file, warn first — that value lands in git unless they add `novps.yaml` to `.gitignore`. Prefer the `${VAR}` + `--env-file` pattern.

After export, show a diff of what was exported, then let the user edit before applying.

### 3. No app exists yet at all?

The user has two modes — pick once, per project:

- **Docker (pre-built image)** — any registry (ghcr, novps registry, Docker Hub). Fastest if they already push images in CI.
- **Build from source (GitHub)** — novps builds from the repo on each deploy. Requires novps GitHub App installed on the repo + the user's GitHub OAuth linked on novps.

Check with `novps github list`. Empty → user must go to the novps dashboard to connect GitHub before build-from-source works. The CLI cannot do that step.

See `reference/novps.docker.example.yaml` and `reference/novps.github.example.yaml`. Schema details in `reference/manifest-schema.md`.

## Fast paths (no manifest edit needed)

Use these when the intent is narrow — skip `apps apply`, go direct.

| Intent | Command |
|---|---|
| Ship a new image tag | `novps resources set-image <resource-id> --tag <sha>` then `novps resources deploy <resource-id>` |
| Change a single env | `novps resources set-env <resource-id> KEY=VALUE` (merges by default) |
| Replace all envs | `novps resources set-env <resource-id> --replace K1=V1 K2=V2` — **confirm with user first** |
| Scale | `novps resources scale <resource-id> -r sm:2` (sizes: `xs` `sm` `md` `lg` `xl`) |
| Restart (redeploy same image) | `novps resources deploy <resource-id>` or `novps apps deploy <app-id>` |
| Edit image + don't deploy yet | `novps resources update <resource-id> --image X --tag Y --no-deploy` |

Default tag strategy when the user doesn't specify: `git rev-parse --short HEAD`.

## Observability

| Intent | Command |
|---|---|
| Tail logs | `novps resources logs <resource-id> -f --since 10m` |
| Search logs | `novps resources logs <resource-id> --since 1h --search "error"` |
| Logs for one pod | `novps resources logs <resource-id> --pod <pod-name>` |
| Shell into pod | `novps resources connect <resource-id>` |
| Resource status | `novps resources get <resource-id>` |
| List app resources | `novps apps resources <app-id>` |
| Port-forward a resource | `novps port-forward resource <resource-id> <remote-port> [-l <local-port>]` |
| Port-forward a database | `novps port-forward database <database-id> [-l <local-port>]` |

When debugging "why is X broken", default to: `resources get` → `resources logs --since 15m --search error` → offer `resources connect` if the user wants to poke inside.

## Resolving IDs from names

The user thinks in names, the CLI wants IDs. Always resolve before acting:

```bash
novps apps list --json                        # find app_id by name
novps apps resources <app-id> --json          # find resource_id by name
novps databases list --json                   # find database_id by name
```

## Databases & storage (read-heavy, some writes)

- `novps databases list|get|create|delete|resize|allow-apps|replica|backups|pool|pg-db|pg-user`
- `novps storage list|create|delete|set-access|files|keys`

For destructive ops (`delete`, `resize`, `set-access`), always show the user what will happen and get explicit confirmation. These are not reversible from the CLI.

## Safety rails

- Never commit real secret values into `novps.yaml`. Use `${VAR}` + `--env-file .env` and ensure `.env` is gitignored.
- `--prune` on `apps apply` deletes resources missing from the manifest — always dry-run first.
- `set-env --replace` wipes existing envs — merge is the default for a reason.
- `databases delete` and `storage delete` are permanent — require explicit user confirmation and echo the target ID back before running.
- Don't run `auth login` with a token the user pastes in plain chat without noting the token is now in session history.

## Common recipes

**"Deploy this repo to novps"** (happy path):
1. Check prereqs.
2. Is there a `novps.yaml`? If no → bootstrap (see above) or ask user if they want to start from an example.
3. `novps apps apply <name> -f novps.yaml --env-file .env --wait`.
4. Tail logs on changed resources until healthy.

**"Ship the latest commit"**:
1. `sha=$(git rev-parse --short HEAD)`, ensure image for that sha exists in registry (or was just pushed).
2. `novps resources set-image <id> --tag $sha`
3. `novps resources deploy <id>`
4. `novps resources logs <id> -f --since 2m` until ready.

**"Something's broken in prod"**:
1. `novps apps resources <app-id>` → find unhealthy resource.
2. `novps resources get <id>` → status, recent events.
3. `novps resources logs <id> --since 15m --search error`.
4. Offer `novps resources connect <id>` for deeper debugging.

**"Connect to the prod db locally"**:
1. `novps databases list` → find id.
2. `novps port-forward database <id> -l 5433`.
3. Print the `psql postgres://...@localhost:5433/...` string using `novps databases get <id>`.

## References

- `reference/manifest-schema.md` — full YAML schema, field by field.
- `reference/novps.docker.example.yaml` — pre-built image starter.
- `reference/novps.github.example.yaml` — build-from-source starter.
- Novps docs: https://docs.novps.io
