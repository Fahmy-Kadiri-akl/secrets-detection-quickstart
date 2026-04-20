# secrets-detection

A dashboard for Akeyless customers to review vault items and decide which
ones to rotate. It imports metadata from the Akeyless CLI, scores each
item on priority and blast radius, and serves the result at
`http://localhost:8000/`.

The container image is published at
`ghcr.io/fahmy-kadiri-akl/secrets_detection`. There is no build step.

## Prerequisites

* Docker 20.10 or newer, or Docker Compose v2.
* The Akeyless CLI installed on the host with at least one named profile.

Check your setup:

```bash
command -v akeyless
ls ~/.akeyless/profiles/*.toml | grep -v '/t-'
```

If the CLI is missing, follow the install guide at
https://docs.akeyless.io/docs/cli. Ephemeral `t-*.toml` token profiles
are ignored; you need a profile you created with `akeyless configure`.

The container does not bundle the CLI. You bind-mount the host binary at
runtime. Image size stays around 168 MB and you run the CLI version you
already trust.

## Run with `docker run`

```bash
docker run --rm -p 8000:8000 \
  -v "$(command -v akeyless):/opt/akeyless:ro" \
  -v "$HOME/.akeyless:/root/.akeyless:ro" \
  -v secdet-data:/data \
  -e SECDET_AKEYLESS_BIN=/opt/akeyless \
  ghcr.io/fahmy-kadiri-akl/secrets_detection:latest
```

Open http://localhost:8000/.

## Run with Docker Compose

```bash
git clone https://github.com/Fahmy-Kadiri-akl/secrets-detection-quickstart
cd secrets-detection-quickstart
AKEYLESS_BIN=$(command -v akeyless) docker compose up
```

Compose refuses to start if `AKEYLESS_BIN` is unset.

## First run

1. The dashboard loads at http://localhost:8000/.
2. In the Import card, pick a profile from the dropdown.
3. Click "Preview account" to confirm which Akeyless account the profile
   is authenticated to. No data is written yet.
4. Click "Import". The container runs `list-items`, `list-roles`,
   `list-targets`, `list-auth-methods`, `list-groups`, `list-gateways`,
   and per-item USC detail lookups. A 200-item tenant takes 10 to 30
   seconds.
5. Open the Inventory tab.

Every row stored in the database carries its `account_id`. Switching
accounts in the banner at the top scopes the whole UI to the selected
account. There is no cross-contamination between tenants.

## Volume mounts

| Mount | Purpose |
|---|---|
| `$(command -v akeyless):/opt/akeyless:ro` | Host Akeyless CLI binary |
| `$HOME/.akeyless:/root/.akeyless:ro` | Host Akeyless profiles (read-only) |
| `secdet-data:/data` | Persistent SQLite database at `/data/secdet.db` |

## Environment variables

| Variable | Default | Notes |
|---|---|---|
| `SECDET_AKEYLESS_BIN` | (none) | Absolute path to the CLI inside the container |
| `SECDET_DB` | `/data/secdet.db` | SQLite database path |
| `SECDET_HOST` | `0.0.0.0` | Web server bind host |
| `SECDET_PORT` | `8000` | Web server bind port |
| `SECDET_LOG_LEVEL` | `INFO` | Python logging level |

## Preflight

The entrypoint validates the environment before starting the web server.
It exits with a non-zero code and a clear message if:

* `akeyless` is not resolvable inside the container.
* `~/.akeyless/profiles/` is missing or contains no named profiles.
* `/data` is not writable.

There are no silent fallbacks. Failures show up in `docker logs`.

## Image tags

| Tag | Use |
|---|---|
| `latest` | Always the most recent release |
| `2026-04-20` | Pinned snapshot: BlastCell signal icons, tier recalibration, clickable Paths and Findings drawers |
| `2026-04-19` | Pinned snapshot: initial publish |
| `0.0.1` | Semver pin to the 2026-04-19 build |

To pin by digest:

```bash
docker pull ghcr.io/fahmy-kadiri-akl/secrets_detection@sha256:01695dc3c2491534a5aba3accd19eeda7abc76c89b86b60f2ce62770891818d4
```

## What the dashboard shows

Overview page:

* Rotation hygiene, median days since rotation, SLA breach rate, and
  percentage of unassigned items.
* Cohort cards for stale, orphan, unassigned, cloud-exposed,
  admin-reachable, high-blast-low-priority, SLA-breach, and never-rotated
  items.
* Import history.

Inventory page (per row):

* Name and path.
* Type.
* Priority tier (P0 to P4) with numeric score.
* Blast radius cell: tier badge, four signal icons (cloud-synced,
  DFC-mitigated, high-impact type, high downstream reach), numeric
  score.
* Lifecycle: age, version, last-access.
* Roles, auth methods, capabilities, tags, targets, cloud environments,
  USC sync count.
* Owner, team, system if configured in Settings.

Click any row, path, or cohort to open a side drawer with the full
breakdown. The drawer exposes the RBAC access graph, sub-claim
constraints on each reachable auth method, and the blast score
components.

Settings page lets you manage folder, tag, and role-substring mappings
that populate owner, team, and system.

## Troubleshooting

| Symptom | Cause |
|---|---|
| `preflight FAILED: akeyless CLI not found` | `SECDET_AKEYLESS_BIN` is unset or `$(command -v akeyless)` returned empty on the host |
| `preflight FAILED: no named akeyless profiles found` | Only ephemeral `t-*.toml` profiles exist. Create a named profile with `akeyless configure --profile myprofile` |
| `preflight FAILED: /data is not writable` | The `/data` mount is read-only. Use a named volume: `-v secdet-data:/data` |
| UI loads, Import returns 502 | The profile's auth token expired. Re-run `akeyless configure --profile <name>` on the host and restart the container |

## Security

* The container reads `~/.akeyless` only. The mount is `:ro`.
* Secret *values* are never pulled. `list-items` and `describe-item`
  return metadata only.
* The SQLite database at `/data/secdet.db` contains item metadata, role
  definitions, auth method configs, and USC sync associations. Treat it
  as sensitive: it exposes your RBAC topology. It does not contain
  vault secrets.

## License

MIT. See [LICENSE](./LICENSE).

The source repository is separate from this quickstart. Contact the
maintainer for access.
