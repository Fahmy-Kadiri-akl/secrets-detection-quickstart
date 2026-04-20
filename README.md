# secrets-detection

A dashboard for Akeyless customers to review vault items and decide which
ones to rotate. It imports metadata from the Akeyless CLI, scores each
item on priority and blast radius, and serves the result at
`http://localhost:8000/`.

The container image is published at
`ghcr.io/fahmy-kadiri-akl/secrets_detection`. There is no build step.

## Prerequisites

* Docker 20.10 or newer (Docker Desktop on macOS or Windows, or Docker
  Engine on Linux).
* At least one named Akeyless profile on the host under
  `~/.akeyless/profiles/` (macOS, Linux, WSL2) or
  `%USERPROFILE%\.akeyless\profiles\` (Windows).

If you don't have a profile yet, install the Akeyless CLI on your host
and run `akeyless configure --profile myprofile`. See
https://docs.akeyless.io/docs/cli.

The container bundles its own Linux build of the Akeyless CLI, so you do
not need the CLI binary on the host. You only need the profile TOML
files so the container can authenticate as you.

## Run with `docker run`

macOS and Linux:

```bash
docker run --rm -p 8000:8000 \
  -v "$HOME/.akeyless:/root/.akeyless:ro" \
  -v secdet-data:/data \
  ghcr.io/fahmy-kadiri-akl/secrets_detection:latest
```

Windows (PowerShell):

```powershell
docker run --rm -p 8000:8000 `
  -v "${env:USERPROFILE}\.akeyless:/root/.akeyless:ro" `
  -v secdet-data:/data `
  ghcr.io/fahmy-kadiri-akl/secrets_detection:latest
```

Windows (cmd):

```cmd
docker run --rm -p 8000:8000 ^
  -v "%USERPROFILE%\.akeyless:/root/.akeyless:ro" ^
  -v secdet-data:/data ^
  ghcr.io/fahmy-kadiri-akl/secrets_detection:latest
```

Open http://localhost:8000/.

## Run with Docker Compose

```bash
git clone https://github.com/Fahmy-Kadiri-akl/secrets-detection-quickstart
cd secrets-detection-quickstart
docker compose up
```

Compose reads `${HOME}` on macOS and Linux. On Windows use `docker
compose up` from a PowerShell session where `$HOME` maps to
`%USERPROFILE%`.

## First run

1. Open http://localhost:8000/.
2. In the Import card, pick a profile from the dropdown.
3. Click "Preview account". The container runs
   `akeyless get-account-settings` using that profile and shows the
   resolved `account_id`. Nothing is written to the database yet.
4. Click "Import". The container pulls `list-items`, `list-roles`,
   `list-targets`, `list-auth-methods`, `list-groups`, `list-gateways`,
   and per-item USC detail. A 200-item tenant takes 10 to 30 seconds.
5. Open the Inventory tab.

Every row stored in the database carries its `account_id`. The account
banner at the top of every page scopes the whole UI to the selected
account, so two accounts imported into the same database do not mix.

## Profile portability

For the common `access_key` auth type, profile TOML files are portable
across macOS, Linux, and Windows. The container reads them as-is.

Profiles that rely on host-specific auth flows (SAML browser redirect,
OAuth2 local callback, Kerberos, cert files at Windows paths) will not
authenticate from inside the container. For those, either:

* Create a second profile that uses API key auth, and use it from this
  container only, or
* Use a short-lived access token (t-token). Run
  `akeyless auth -a <access-id> -k <access-key>` on the host to mint
  one, then paste it into the CLI's `--auth-token` flag. A UI option
  for pasting a t-token is on the roadmap.

## Volume mounts

| Mount | Purpose |
|---|---|
| `$HOME/.akeyless:/root/.akeyless:ro` | Host Akeyless profiles (read-only) |
| `secdet-data:/data` | Persistent SQLite database at `/data/secdet.db` |

## Environment variables

| Variable | Default | Notes |
|---|---|---|
| `SECDET_AKEYLESS_BIN` | `/opt/akeyless/akeyless` | Absolute path to the bundled CLI. Override only if you mount a different binary. |
| `SECDET_DB` | `/data/secdet.db` | SQLite database path |
| `SECDET_HOST` | `0.0.0.0` | Web server bind host |
| `SECDET_PORT` | `8000` | Web server bind port |
| `SECDET_LOG_LEVEL` | `INFO` | Python logging level |

## Preflight

The entrypoint validates the environment before starting the web server.
It exits with a non-zero code and a clear message if:

* The bundled Akeyless CLI is missing or not executable.
* `~/.akeyless/profiles/` is missing or contains no named profiles.
* `/data` is not writable.

Failures appear in `docker logs`.

## Image tags

| Tag | Use |
|---|---|
| `latest` | Always the most recent release |
| `2026-04-20-bundled` | Pinned snapshot: bundles the Akeyless Linux CLI so no host binary is needed |
| `2026-04-20` | Pinned snapshot: BlastCell signal icons, tier recalibration, clickable drawers (requires host CLI mount) |
| `2026-04-19` | Initial publish (requires host CLI mount) |
| `0.0.1` | Semver pin to 2026-04-19 |

Pin by digest:

```bash
docker pull ghcr.io/fahmy-kadiri-akl/secrets_detection@sha256:9db5a763e3792e584ad619b0101234abb97cf7e1ec304712c862792d69df77d6
```

## Troubleshooting

| Symptom | Cause |
|---|---|
| `preflight FAILED: no named akeyless profiles found` | Only ephemeral `t-*.toml` token profiles exist. Create a named profile with `akeyless configure --profile myprofile` on the host |
| `preflight FAILED: /data is not writable` | The `/data` mount is read-only. Use a named volume: `-v secdet-data:/data` |
| Import returns 502 with `exit 1` | The profile's auth flow does not work from inside the container (SAML, OAuth, or host-cert). See "Profile portability" above |
| Import returns 502 with `token expired` | The profile's access token expired. Re-authenticate on the host: `akeyless configure --profile <name>` |

## Security

* The container reads `~/.akeyless` only. The mount is `:ro`.
* The Akeyless CLI `list-*` and `describe-*` commands return metadata,
  not secret values.
* The SQLite database at `/data/secdet.db` contains item metadata, role
  definitions, auth method configs, and USC sync associations. Treat
  it as sensitive: it exposes your RBAC topology. It does not contain
  vault secrets.

## License

MIT. See [LICENSE](./LICENSE).

The source repository is separate from this quickstart. Contact the
maintainer for access.
