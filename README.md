# Zablo GitHub Action

[![PyPI](https://img.shields.io/pypi/v/zablo-cli.svg)](https://pypi.org/project/zablo-cli/)
[![License](https://img.shields.io/badge/license-Apache--2.0-blue.svg)](LICENSE)

Fetch zero-knowledge secrets from a [Zablo](https://github.com/r2l332/zablo)
server and inject them into your GitHub Actions workflow — without ever
storing a long-lived API key in `Settings → Secrets`.

- 🔐 **OIDC federation by default** — your runner mints a short-lived GitHub
  ID token, Zablo exchanges it for a `vks_…` session, and the session is used
  for the run only.
- 🕶 **Zero-knowledge** — the passphrase lives in a GitHub secret; the
  server never sees plaintext or the derived key.
- 🎭 **Automatic log masking** — every fetched value (line-by-line, so PEM
  keys work too) is masked via `::add-mask::`.
- 🧰 **Composite action** — no Docker pull, no Node runtime; just bash +
  [`zablo-cli`](https://pypi.org/project/zablo-cli/) installed via `pipx`.

## Quickstart

```yaml
permissions:
  id-token: write        # required for OIDC federation
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: r2l332/zablo-action@v1
        with:
          api-url: https://zablo.example.com
          passphrase: ${{ secrets.ZABLO_PASSPHRASE }}
          secrets: |
            DATABASE_URL=prod/db/url
            STRIPE_KEY=prod/stripe/live
            REDIS_PASSWORD=prod/redis/pw

      - name: Deploy
        run: ./deploy.sh
        # $DATABASE_URL / $STRIPE_KEY / $REDIS_PASSWORD are populated
```

## Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `api-url` | ✅ | — | Zablo server URL. |
| `passphrase` | ✅ if `secrets` set | — | Client-side encryption passphrase. Store as a GitHub secret. |
| `secrets` | | — | Multiline `NAME=path` list. Empty = install CLI only. |
| `api-key` | | — | Static `vk_…` key. If unset, action federates via OIDC. |
| `audience` | | `zablo.io` | OIDC audience. |
| `version` | | `latest` | zablo-cli version (e.g. `0.4.0`). |

## Outputs

| Name | Description |
|------|-------------|
| `cli-version` | Installed zablo-cli version. |

## Modes

### 1. OIDC federation (recommended)

The runner exchanges its GitHub ID token for a Zablo session. **No API key
in GitHub Secrets required** — the only secret is the passphrase (which
never leaves the runner).

```yaml
permissions:
  id-token: write
  contents: read

steps:
  - uses: r2l332/zablo-action@v1
    with:
      api-url: https://zablo.example.com
      passphrase: ${{ secrets.ZABLO_PASSPHRASE }}
      secrets: |
        DATABASE_URL=prod/db/url
```

Requires a matching federation binding on the Zablo server that maps the
GitHub OIDC claims (`repository`, `ref`, etc.) to a machine user.

### 2. Static API key (fallback)

Simpler to set up, but you're back to a long-lived credential in GitHub Secrets.

```yaml
steps:
  - uses: r2l332/zablo-action@v1
    with:
      api-url: https://zablo.example.com
      api-key: ${{ secrets.ZABLO_API_KEY }}
      passphrase: ${{ secrets.ZABLO_PASSPHRASE }}
      secrets: |
        DATABASE_URL=prod/db/url
```

### 3. Install-only

Just install `zablo-cli` on the runner; do everything by hand afterwards.

```yaml
steps:
  - uses: r2l332/zablo-action@v1
    with:
      api-url: https://zablo.example.com
  - run: |
      zablo --help
      # ... custom flow
```

## Security notes

- **Every fetched value is masked** in the log via `::add-mask::` before it
  reaches `$GITHUB_ENV`. Multi-line values are masked line-by-line.
- **The session token is short-lived** (Zablo default: ~1 hour) — even if
  the log is leaked, replay is time-bound.
- **The passphrase is not sent to the server.** Ever. It's used to derive
  the AES-256-GCM key locally on the runner.
- **Pin the action to a SHA in production** (`r2l332/zablo-action@abc1234`)
  and check what the composite steps do (this is a bash-only action, no
  hidden binaries).

## Development

Local test with [`act`](https://github.com/nektos/act) or plain `bash`:

```bash
bash -n action.yml       # syntax check
# For real testing, push to a test repo and trigger the workflow.
```

## License

Apache-2.0
