# novps.yaml manifest schema

The manifest is consumed by `novps apps apply <app-name> -f novps.yaml`. The app name is the CLI argument, not a field in the YAML.

## Top level

| Field | Type | Notes |
|---|---|---|
| `envs` | `list<{key,value}>` | App-level environment variables. Merged into every resource; resource-level `envs` override on conflict. |
| `resources` | `list<resource>` | One entry per deployable unit (web-app, worker, cron-job). |

## Resource

| Field | Type | Notes |
|---|---|---|
| `name` | string | Unique within the app. Used to match resources across applies. |
| `type` | `web-app` \| `worker` \| `cron-job` | Determines which `config` fields are meaningful. |
| `source_type` | `docker` \| `github` | How novps gets the image. |
| `source` | object | Shape depends on `source_type`. See below. |
| `config` | object | Runtime config. See below. |
| `replicas` | `{type, count}` | `type`: `xs` \| `sm` \| `md` \| `lg` \| `xl`. `count`: integer. |
| `envs` | `list<{key,value}>` | Resource-level env vars. Use `${VAR}` for `--env-file` substitution. |
| `volumes` | `list` | Persistent volume mounts. Always include the field — use `[]` when no volumes are needed. |

## `source` — docker

Pull a pre-built image from any registry.

```yaml
source_type: docker
source:
  type: docker
  credentials: ''              # registry credentials ref (empty = public/anonymous)
  name: ghcr.io/org/repo       # full image path
  tag: latest                  # image tag (prefer a git sha in CI)
```

## `source` — github (build from source)

Novps builds the image on each deploy. Requires:
- novps GitHub App installed on the repo
- user's GitHub OAuth linked on novps

Verify with `novps github list` — empty means the integration isn't set up yet, and the CLI cannot fix it (dashboard only).

```yaml
source_type: github
source:
  type: github
  repository: owner/repo
  branch: main
  source_dir: ./               # path inside the repo (e.g. ./backend for monorepos)
  build_command: docker build .
  build_envs:                  # build-time env (optional)
    - key: NODE_ENV
      value: production
```

## `config`

| Field | Applies to | Notes |
|---|---|---|
| `command` | worker, cron-job | Startup command override. Empty = use image default. |
| `port` | web-app | HTTP port the app listens on (string, e.g. `"8080"`). |
| `internal_ports` | any | Additional exposed ports. |
| `schedule` | cron-job | Cron expression (e.g. `* * * * *`). |
| `allow_overlapping` | cron-job | Whether runs can overlap. |
| `restart_policy` | any | Known values: `always`, `never`. Use `novps apps export` on an existing app to see other values in use. |

## Env var substitution

Values like `${DB_URL}` are resolved at apply time from:
1. The shell environment
2. A `.env` file passed via `--env-file`

This is how secrets stay out of the manifest. Commit the YAML; keep `.env` gitignored.

## Example — minimal web-app from GitHub

```yaml
envs:
  - key: LOG_LEVEL
    value: info

resources:
  - name: api
    type: web-app
    source_type: github
    source:
      type: github
      repository: owner/repo
      branch: main
      source_dir: ./
      build_command: docker build .
    config:
      port: "8080"
      restart_policy: always
    replicas:
      type: sm
      count: 2
    envs:
      - key: DB_URL
        value: ${DB_URL}
    volumes: []
```

## Example — worker from docker image

```yaml
resources:
  - name: worker
    type: worker
    source_type: docker
    source:
      type: docker
      credentials: ''
      name: ghcr.io/org/repo
      tag: v1.2.3
    config:
      command: "python -m app.worker"
      restart_policy: always
    replicas:
      type: sm
      count: 1
    envs: []
    volumes: []
```
