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

## Step 0 — Project discovery (run before asking anything)

A good deploy UX means **investigating first, asking last**. Before any question to the user, inspect the working directory and propose a plan they can accept or tweak. Never dump a list of questions cold.

Run this discovery sequence end-to-end, then summarise findings and show a draft manifest:

1. **Git / GitHub signals**
   - `git rev-parse --is-inside-work-tree` — is this a git repo?
   - `git remote get-url origin` — is there a GitHub remote? (Enables build-from-source.)
   - `git rev-parse --short HEAD` — default image tag when git is available.
2. **Service topology**
   - `docker-compose.yml` / `docker-compose.yaml` / `compose.yaml` — **authoritative** when present. Every service becomes a candidate resource. Read `image`, `build`, `command`, `ports`, `environment`.
   - `Procfile` — `web:` → web-app, `worker:` → worker, `release:` → one-off (pre-deploy).
   - `Dockerfile` — single-service fallback. Read `EXPOSE` for port, `CMD`/`ENTRYPOINT` for command hints.
3. **Framework conventions** (only after the above — don't invent services)
   - **Laravel** (`composer.json` contains `laravel/framework`): typical resources are `web` (nginx/php-fpm), `horizon` or `queue` worker (`php artisan queue:work` / `horizon`), `scheduler` cron (`* * * * *` → `php artisan schedule:run`).
   - **Django** (`requirements.txt` or `pyproject.toml` has `django`): `web` (gunicorn/uvicorn), plus `celery` + `celery-beat` workers if celery is present.
   - **Rails**: `web` (puma), `sidekiq` worker if `sidekiq` in Gemfile.
   - **Next.js / Node**: single `web` from `npm start`, unless `package.json` scripts hint at workers.
4. **Data services** — postgres / redis detected as compose services?
   - Novps doesn't auto-provision from the manifest. The skill **creates managed instances** by default (don't ask whether to provision — ask *which size*, only if non-trivial).
   - Supported engines: `postgres` (versions 14, 15, 16), `redis`. For anything else (mysql, mongo, clickhouse), flag it — user must either run it as a resource in the manifest or point to an external host.
   - See the "Provisioning databases" section below for the exact flow.
5. **Existing envs**
   - If `.env` or `.env.example` exists, parse keys and mirror them into the manifest's `envs` as `${KEY}` placeholders. Don't copy values.

### Discovery output — what to present to the user

After discovery, output exactly this:

```
Detected: <stack>, <N> services: <names>
Source mode: <github | docker> because <reason>
Manifest draft written to novps.yaml (review before apply)
```

Then show the draft and ask a single yes/no: *"Apply this as `<app-name>`?"* If the user rejects specific parts, edit in place rather than re-asking from scratch.

## Choosing `source_type` — decide, don't ask

```
Has GitHub remote (git remote get-url origin → github.com/...)?
├── Yes
│   └── novps github list → non-empty?
│       ├── Yes → source_type: github  ← DEFAULT for new projects
│       └── No  → tell user: "Install the novps GitHub App on this repo
│                 in the novps dashboard, then rerun. Or pick docker mode."
└── No
    └── source_type: docker
        ├── novps registry list → has a namespace?
        │   ├── Yes → use it (single auth, no cross-registry creds)
        │   └── No  → direct user to create a namespace in the novps dashboard
        └── Or any external registry the user already uses (ghcr, Docker Hub).
```

**Why github-source is the default for new projects:** zero registry setup, zero local build, no credentials to juggle. Only fall back to docker mode when the repo isn't on GitHub, the user already has a CI that pushes images, or the GitHub App isn't installable.

**Default image tag:**
- First deploy (no previous image): `latest`.
- Subsequent deploys in a git repo: `$(git rev-parse --short HEAD)`.
- Only ask the user if they explicitly want to pick the tag.

## Provisioning databases (postgres / redis)

When discovery detects a `postgres` or `redis` service in `docker-compose.yml` (or via framework conventions like Laravel needing a DB), the skill **provisions the managed instance automatically** as part of the deploy flow — don't stop and ask whether to create it.

### Flow

1. **Create** (default size `sm`, bump only if the user said "prod" or similar):
   ```bash
   novps databases create --engine postgres -s sm --postgres-version 16 --wait --json
   novps databases create --engine redis    -s sm --wait --json
   ```
   Parse the returned `id` from JSON.
2. **Read credentials** in env format:
   ```bash
   novps databases get <db-id> --format env --show-password
   ```
   Append the output lines directly into the repo's `.env` (creating it if missing). This gives `DATABASE_URL`, `REDIS_URL`, etc. — whatever the CLI emits, pass through as-is, don't rename.
3. **Grant the app access**:
   ```bash
   novps databases allow-apps <db-id> <app-name>
   ```
   Required — databases are isolated by default. Do this before `apps apply`, or the first deploy will come up unable to connect.
4. **Mirror into the manifest**: ensure the resource `envs` reference the keys as `${DATABASE_URL}`, `${REDIS_URL}`, etc. Apply picks them up via `--env-file .env`.

### Defaults

- Size: `sm` for new/dev apps. Ask only if the user mentions production/scale.
- Postgres version: `16` (latest supported) unless the project pins a version in compose or docs.
- Node count: `1` — replicas are a separate decision the user should make explicitly.

### When *not* to auto-provision

- An existing novps database with a matching name is already in `databases list` — use it, don't create a duplicate. Offer the user: "found existing `<name>` postgres, wire to it? (y/n)".
- The user has external managed DBs (RDS, Neon, Upstash) — let them paste the URL, don't create anything.
- The engine isn't supported (`mysql`, `mongo`, `clickhouse`) — tell the user, suggest either self-hosting as a resource or external provider.

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

Run **Step 0 — Project discovery** (above), pick `source_type` via the decision tree, draft a `novps.yaml` from detected services, show it to the user, then apply. See `reference/novps.docker.example.yaml` and `reference/novps.github.example.yaml` for starter shapes; schema in `reference/manifest-schema.md`.

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

Default tag strategy: `latest` for a first deploy, `$(git rev-parse --short HEAD)` once the project has a git repo and an image has been shipped before.

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
2. Is there a `novps.yaml`? If yes → skip to step 6.
3. **Project discovery** (Step 0): inspect git remote, `docker-compose.yml` / `Procfile` / `Dockerfile`, framework files. Pick `source_type` via the decision tree.
4. **Provision data services** detected in compose (postgres/redis) — `databases create --wait`, append `databases get --format env --show-password` output to `.env`, `databases allow-apps`.
5. Draft `novps.yaml` from detected application services (web/worker/cron as inferred), referencing `.env` keys as `${VAR}`. Show it, get one yes/no.
6. `novps apps apply <name> -f novps.yaml --env-file .env --wait`.
7. Tail logs on changed resources until healthy.

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
