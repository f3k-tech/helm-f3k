# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository overview

This is a private Helm chart repository for F3K. Charts are published via GitHub Pages on the `gh-pages` branch. The `main` branch contains chart source; `gh-pages` holds the packaged `.tgz` files and the generated `index.yaml`.

## Common commands

```bash
# Lint a chart
helm lint charts/<chart-name>

# Render templates locally (dry-run)
helm template my-release charts/<chart-name> -f charts/<chart-name>/values.yaml

# Package a chart
helm package charts/<chart-name>

# Update chart dependencies (needed after editing Chart.yaml)
helm dependency update charts/<chart-name>
```

## Charts

| Chart | Purpose |
|---|---|
| `api-platform` | Symfony / API Platform PHP app — the most complex chart |
| `basic-app` | Generic single-container app |
| `mediamtx` | BluEnviron MediaMTX streaming server |
| `panorama-server` | 360° panorama serving app |

## api-platform architecture

The chart deploys a Symfony/API Platform app and supports three database backends (one enabled at a time):

- **MariaDB** (`mariadb.enabled`) — uses the Bitnami MariaDB subchart
- **PostgreSQL** (`postgresql.enabled`) — uses CloudNativePG (CNPG) operator via a `Cluster` CRD
- **SQLite** (`persistence.sqlite.enabled`) — backed by a PVC

### Secret management pattern

Secrets are stable across Helm upgrades via a `lookup` guard: if a secret already exists in the cluster, its current value is preserved; otherwise a new value is generated. This pattern is used in:

- `templates/secret.yaml` — app secrets (`app-secret`, `jwt-passphrase`, `cron-secret`)
- `templates/database/secret.yaml` — postgres credentials (`username`, `password`)

The `cron-secret` key is stored alongside the app secrets in the `-app` secret.

### JWT keys

A pre-install/pre-upgrade `Job` (`job-jwt-keys-secret-generator.yaml`) generates an RSA key pair and stores it in a `<fullname>-jwt-keys` secret if one does not yet exist. The `role-secret-creator` Role + RoleBinding grant the job the necessary RBAC to create secrets.

### PostgreSQL DATABASE_URL construction

When `postgresql.enabled`, the deployment injects `POSTGRES_USER` and `POSTGRES_PASSWORD` from the CNPG secret (`<fullname>-postgres`), then constructs `DATABASE_URL` using Kubernetes env-var interpolation (`$(POSTGRES_USER)`). The CNPG read-write service is `<fullname>-postgres-rw`. Override the version string via `postgresql.serverVersion` (defaults to `"16"`).

### Helper naming

All named templates live in `templates/_helpers.tpl`:
- `api-platform.fullname` — base name for most resources
- `api-platform.fullname-postgres` — name for the CNPG Cluster and its secret (`<fullname>-postgres`)

## Publishing charts

Charts are published by packaging them and updating `gh-pages`. The `index.yaml` on `gh-pages` is the Helm repository index. The chart registry URL is `https://f3k-tech.github.io/helm-f3k/`.
