# secrets-detection — quickstart

Visualize which **Akeyless** vault secrets should be rotated next.

This repo is the quickstart for running the pre-built container; the
application ingests `akeyless list-items` output from your tenant, scores
each secret by rotation priority + blast radius, and serves a dashboard
at http://localhost:8000/.

The image is published publicly at
**`ghcr.io/fahmy-kadiri-akl/secrets_detection`**. No build step required.

---

## Prerequisites

1. **Docker** (`docker >= 20.10`) or **Docker Compose v2**.
2. The **Akeyless CLI** installed on the host and configured with at least
   one named profile:
   ```bash
   command -v akeyless   # must resolve to a binary
   ls ~/.akeyless/profiles/*.toml | grep -v '/t-'   # at least one named profile
   ```
   If you don't have it yet, follow
   [docs.akeyless.io/docs/cli](https://docs.akeyless.io/docs/cli) to install
   and configure a profile.

The container does **not** bundle the CLI — you bind-mount your host's
binary into the container. This keeps the image small (~168 MB) and means
you're always running the exact CLI version you trust.

---

## Run it — one command

```bash
docker run --rm -p 8000:8000 \
  -v "$(command -v akeyless):/opt/akeyless:ro" \
  -v "$HOME/.akeyless:/root/.akeyless:ro" \
  -v secdet-data:/data \
  -e SECDET_AKEYLESS_BIN=/opt/akeyless \
  ghcr.io/fahmy-kadiri-akl/secrets_detection:latest
```

Then open **http://localhost:8000/**.

## Run it — Docker Compose

```bash
git clone https://github.com/Fahmy-Kadiri-akl/secrets-detection-quickstart
cd secrets-detection-quickstart
AKEYLESS_BIN=$(command -v akeyless) docker compose up
```

Compose refuses to start if `AKEYLESS_BIN` is unset — no silent fallback.

---

## First-run flow

1. Dashboard loads at http://localhost:8000/.
2. In the **Import** card, pick a profile from the dropdown.
3. Click **Preview account** — confirms which Akeyless account this
   profile is authenticated to (e.g. `acc-abc123…`). No data is written
   to the DB until you click Import.
4. Click **Import** — pulls `list-items`, `list-roles`, `list-targets`,
   `list-auth-methods`, `list-groups`, `list-gateways`, and per-item USC
   details via the CLI. A 200-item tenant typically takes 10–30 seconds.
5. Go to **Inventory** — the rotation-priority dashboard.

Every imported row is stamped with its `account_id`. Switching accounts
(in the banner at the top of every page) scopes the entire UI to the
selected account — no cross-contamination between tenants.

---

## Runtime contract

| Volume mount                               | Purpose                                          |
|--------------------------------------------|--------------------------------------------------|
| `$(command -v akeyless):/opt/akeyless:ro`  | Host's akeyless CLI binary                       |
| `$HOME/.akeyless:/root/.akeyless:ro`       | Host's akeyless profiles (read-only)             |
| `secdet-data:/data`                        | Persistent SQLite DB at `/data/secdet.db`        |

| Environment variable   | Default           | Notes                                     |
|------------------------|-------------------|-------------------------------------------|
| `SECDET_AKEYLESS_BIN`  | —                 | Absolute path to the CLI inside container |
| `SECDET_DB`            | `/data/secdet.db` | SQLite DB path                            |
| `SECDET_HOST`          | `0.0.0.0`         | Bind host for the web server              |
| `SECDET_PORT`          | `8000`            | Bind port for the web server              |
| `SECDET_LOG_LEVEL`     | `INFO`            | Python logging level                      |

### Preflight

The container's entrypoint validates the environment before starting
uvicorn, and **exits non-zero with a clear message** if:

- `akeyless` isn't resolvable inside the container
- `~/.akeyless/profiles/` is empty or missing (no named profiles)
- `/data` is not writable

No fallbacks. If something's wrong, `docker logs` tells you exactly what.

---

## Image tags

| Tag          | Use                                                        |
|--------------|------------------------------------------------------------|
| `latest`     | Rolling — always the most recent release                   |
| `2026-04-20` | Date-pinned (BlastCell signal icons + tier recalibration)  |
| `2026-04-19` | Date-pinned (initial publish)                              |
| `0.0.1`      | Pinned semver — pins to the 2026-04-19 build               |

Pull by digest if you need byte-for-byte reproducibility:

```bash
docker pull ghcr.io/fahmy-kadiri-akl/secrets_detection@sha256:fa09ea4b721bbe92968b8e8ae2376ca56c3ef5bf07a4236560113ef797901636
```

---

## What's in the dashboard

- **Overview** — hygiene KPIs (% rotated, median days since rotation, SLA
  breach %, % unassigned), cohort cards, and an import history table.
- **Inventory** — the 200-row table. Each row shows:
  - Name + path
  - Type
  - Priority tier (P0..P4) + score
  - **Blast radius cell** — tier badge + 4 signal icons (cloud-synced,
    DFC-mitigated, high-impact type, high downstream reach) + score
  - Lifecycle (age · version · last-accessed)
  - Roles / Auth methods / Capabilities / Tags / Targets / Cloud / USC
  - Owner / Team / System (from your configurable mappings)
- **Settings** — manage the folder / tag / role-substring mappings that
  populate owner / team / system fields.

Click any row to open the detail drawer: full RBAC access graph, sub-claim
constraints on each reachable auth method, and the full blast-radius
breakdown.

---

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| `preflight FAILED: akeyless CLI not found` | `AKEYLESS_BIN` env var unset, or `$(command -v akeyless)` didn't resolve on the host |
| `preflight FAILED: no named akeyless profiles found` | Only ephemeral `t-*.toml` token profiles exist; configure a named profile: `akeyless configure --profile myprofile` |
| `preflight FAILED: /data is not writable` | Volume mount is read-only; use `-v secdet-data:/data` (named volume) |
| UI loads but Import returns 502 | Profile's token has expired — re-run `akeyless configure --profile <name>` and restart the container |

---

## Security model

- The container **only reads** from `~/.akeyless` (the mount is `:ro`).
- Secret *values* are never pulled — only metadata from `list-items`,
  `describe-item`, etc. `list-items` does not return secret material.
- The SQLite DB at `/data/secdet.db` contains item metadata, role
  definitions, auth method configs, and USC sync associations. Treat it
  as sensitive (it reveals your RBAC topology) but no vault secrets.

---

## License

MIT — see [LICENSE](./LICENSE).

The source for the container lives in a separate repository; contact the
maintainer for access.
