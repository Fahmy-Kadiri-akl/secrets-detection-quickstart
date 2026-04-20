# secrets-detection

A dashboard for Akeyless customers to review vault items and decide
which ones to rotate. It imports metadata from the Akeyless CLI, scores
each item on priority and blast radius, and serves the result at
`http://localhost:8000/`.

The container image is published at
`ghcr.io/fahmy-kadiri-akl/secrets_detection:latest`. There is no build
step.

## Prerequisites

* Docker 20.10 or newer (Docker Desktop on macOS or Windows, or Docker
  Engine on Linux).
* A short-lived Akeyless token (`t-...`) from your tenant. Mint it
  however you prefer: the Akeyless web console, the Akeyless REST
  API (`POST /auth`), or the Akeyless CLI on any machine.

The container authenticates each request with the token you paste into
the UI. It does not read any host credentials or profiles.

## Run with `docker run`

Same command for macOS, Linux, and Windows:

```bash
docker run --rm -p 8000:8000 \
  -v secdet-data:/data \
  ghcr.io/fahmy-kadiri-akl/secrets_detection:latest
```

Open http://localhost:8000/.

## Run with Docker Compose

```bash
git clone https://github.com/Fahmy-Kadiri-akl/secrets-detection-quickstart
cd secrets-detection-quickstart
docker compose up
```

## First run

1. Mint a short-lived token from your tenant. Any of:

   * **Akeyless web console**: Settings, CLI / API tokens (or equivalent
     in your tenant).
   * **Akeyless REST API**: `POST /auth` with your credentials returns
     `{"token": "t-..."}`.
   * **Akeyless CLI** on any machine where it is installed:
     ```bash
     akeyless auth -a <access-id> -k <access-key>   # API key
     akeyless auth -a <access-id>                   # SAML / OAuth
     ```

   Copy the resulting `t-xxxxxxxxxxxx` string.

2. Open http://localhost:8000/.

3. In the Import card, paste the t-token into the token field.

4. Click **Preview account**. The container calls
   `akeyless get-account-settings --profile <token>` and shows the
   resolved `account_id`. Nothing is written to the database.

5. Click **Import**. The container pulls `list-items`, `list-roles`,
   `list-targets`, `list-auth-methods`, `list-groups`, `list-gateways`,
   and per-item USC detail. A 200-item tenant takes 10 to 30 seconds.

6. Open the Inventory tab.

The token is cleared from the UI after a successful import. Every row
stored in the database carries its `account_id`. The account banner at
the top of every page scopes the UI to the selected account, so two
imports from different accounts do not mix.

## Volume mounts

| Mount | Purpose |
|---|---|
| `secdet-data:/data` | Persistent SQLite database at `/data/secdet.db` |

No host bind mounts are required.

## Environment variables

| Variable | Default | Notes |
|---|---|---|
| `SECDET_AKEYLESS_BIN` | `/root/.akeyless/bin/akeyless` | Absolute path to the bundled CLI. Override only to mount a different binary. |
| `SECDET_DB` | `/data/secdet.db` | SQLite database path |
| `SECDET_HOST` | `0.0.0.0` | Web server bind host |
| `SECDET_PORT` | `8000` | Web server bind port |
| `SECDET_LOG_LEVEL` | `INFO` | Python logging level |

## Image

Only one tag is supported:

| Tag | Notes |
|---|---|
| `latest` | The current release |

Pin by digest if you need byte-for-byte reproducibility:

```bash
docker pull ghcr.io/fahmy-kadiri-akl/secrets_detection@sha256:a3df99308cef395fa25994bf5e5ce56ac3f62b338a745b38036c7a1bc7ed65d9
```

## Preflight

The entrypoint validates the environment before starting the web server.
It exits with a non-zero code and a clear message if:

* The bundled Akeyless CLI is missing or not executable.
* `/data` is not writable.

Failures appear in `docker logs`.

## Troubleshooting

| Symptom | Cause |
|---|---|
| `auth_token is required` (400) | The token field is empty. Paste a `t-xxx` string. |
| `resolve auth failed (502)` with `exit 1` | The token is expired or invalid. Mint a fresh one. |
| Import returns 502 partway | Most likely an expired token or a transient Akeyless gateway error. Retry with a fresh token. |

## Security

* The container does not read any host credentials directly.
* T-tokens are short-lived (typically 1 to 6 hours). The UI clears the
  token from memory after a successful import.
* The SQLite database at `/data/secdet.db` contains item metadata, role
  definitions, auth method configs, and USC sync associations. Treat
  it as sensitive: it exposes your RBAC topology. It does not contain
  vault secrets.

## License

MIT. See [LICENSE](./LICENSE).

The source repository is separate from this quickstart. Contact the
maintainer for access.
