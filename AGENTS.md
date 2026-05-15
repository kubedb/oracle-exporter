# AGENTS.md - Oracle AI Database Metrics Exporter (KubeDB fork)

This file provides instructions for AI coding agents working in this Oracle Database Prometheus exporter repository.

## Project Overview

Prometheus/OTEL metrics exporter for Oracle AI Database. This is KubeDB's fork of upstream [oracle/oracle-db-appdev-monitoring](https://github.com/oracle/oracle-db-appdev-monitoring) (Go module path: `github.com/oracle/oracle-db-appdev-monitoring`), published as `ghcr.io/kubedb/oracle-exporter`. Built in Go 1.24, uses CGO with Oracle Instant Client via the `godror` driver by default and supports a pure-Go fallback via `go-ora` (`goora` build tag). Loads metrics for one or more databases from a YAML config (`--config.file`) and metric definitions from TOML files (default `default-metrics.toml` plus user-supplied custom metrics). Exposes Prometheus metrics on port `9161` and optionally tails the Oracle alert log to JSON.

## Build & Development Commands

```bash
# Print version (from Makefile VERSION variable)
make version

# Local Go build (uses TAGS=godror by default; requires Oracle Instant Client + CGO)
make go-build

# Cross-compile targets
make go-build-linux-amd64
make go-build-linux-arm64
make go-build-darwin-amd64
make go-build-darwin-arm64
make go-build-windows-amd64

# Build all linux artifacts
make dist

# Run tests (uses goora build tag so no Oracle client is required)
make go-test

# Lint via dockerized golangci-lint v1.50.1
make go-lint

# Single-platform Docker builds
make docker            # alias for docker-amd
make docker-amd
make docker-arm

# Multi-arch buildx + push (used by CI; pushes to ghcr.io/kubedb/oracle-exporter)
make docker-release

# Podman multi-arch build/push
make podman-build
make podman-push
make podman-release

# Cleanup
make clean
```

### Build tags

- `godror` (default) - uses Oracle's `github.com/godror/godror` driver, requires CGO and Oracle Instant Client at runtime. Selected via `collector/connect_godror.go` (`//go:build godror`).
- `goora` - pure-Go driver `github.com/sijms/go-ora/v2`, no CGO needed. Selected via `collector/connect_goora.go` (`//go:build goora`). Used by `make go-test`.

Switch driver via `TAGS=goora make go-build` or `DOCKER_TARGET=exporter-goora make docker`.

## Project Structure

```
main.go                            # Entry point: kingpin flags, exporter wiring, HTTP server, alert-log ticker
collector/                         # Core exporter package
  collector.go                     # Exporter (implements prometheus.Collector), scrape orchestration
  config.go                        # MetricsConfiguration, DatabaseConfig, VaultConfig, LoggingConfig (YAML)
  database.go                      # Database struct, connection-pool warmup, IsValid/invalidate, UpMetric
  connect_godror.go                # godror-tagged connect() implementation
  connect_goora.go                 # goora-tagged connect() implementation
  data_loader.go                   # Scrape gating: isScrapeMetric, getScrapeInterval, getQueryTimeout
  metrics.go                       # Per-metric Prometheus value generation, label/type handling
  cache.go                         # MetricsCache for per-metric custom scrape intervals
  types.go                         # Exporter, Database, Metric, MetricsCache, Config structs
  default_metrics.go               # Loads default metrics TOML
  default_metrics.toml             # Default Oracle metric definitions (queries)
  errors.go                        # Sentinel errors
  collector_test.go                # duplicatedLabels tests
  database_test.go                 # IsValid / invalidate tests
alertlog/alertlog.go               # Tails alert log per-database, emits JSON LogRecord
azvault/azvault.go                 # Azure Key Vault credential loader
ocivault/ocivault.go               # OCI Vault credential loader
hashivault/hashivault.go           # HashiCorp Vault credential loader
kubernetes/                        # Sample Kubernetes manifests (ConfigMap, Deployment, Service, ServiceMonitor)
docker-compose/                    # Local stack: exporter, oracle, prometheus, grafana, txeventq-load
custom-metrics-example/            # Sample TOML custom-metric files
default-metrics.toml               # Top-level copy bundled with binary at /default-metrics.toml
default-metrics.yaml               # YAML mirror of default metrics
example-config.yaml                # Single-database config example
example-config-multi-database.yaml # Multi-database config example
tests/1000-integers.toml           # Test fixture
site/                              # Docusaurus documentation site
docs/                              # Pre-built static docs
Dockerfile                         # Multi-stage: build -> exporter-godror / exporter-goora
Makefile                           # Build, lint, test, docker, podman targets
.github/workflows/                 # ci.yml (PR/master build via `make release`), release.yml
```

## Key Packages / APIs

- `collector.NewExporter(logger, *MetricsConfiguration) *Exporter` - constructs the exporter; warms connection pools concurrently.
- `Exporter` implements `prometheus.Collector` (`Describe`, `Collect`) and exposes `ScrapeInterval()`, `RunScheduledScrapes(ctx)`, `GetDBs()`.
- `collector.LoadMetricsConfiguration(logger, *Config, metricPath, webFlags) (*MetricsConfiguration, error)` - parses `--config.file` YAML, applies env-var expansion, merges with CLI/env defaults.
- `collector.NewDatabase(logger, dblabel, dbname, DatabaseConfig) *Database` - calls driver-specific `connect()`; never panics on bad config (validation deferred to `Ping`).
- `collector.Database.UpMetric(labels) prometheus.Metric` - the `oracledb_up` gauge per database.
- `alertlog.UpdateLog(destination, logger, *collector.Database)` - polls `V$DIAG_ALERT_EXT` and appends JSON records.
- Vault adapters expose `GetSecret(...)` style helpers consumed by `DatabaseConfig.GetUsername()` / `GetPassword()` in `collector/config.go`.

Prometheus namespace is `oracledb`; built-in metric `oracledb_up{database="..."}` plus all custom-defined contexts.

## Configuration

YAML config (`--config.file`) shape (see `example-config.yaml`, `example-config-multi-database.yaml`):

```yaml
databases:
  default:
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    url: localhost:1521/freepdb1
    queryTimeout: 5
    maxOpenConns: 10
    maxIdleConns: 10
    # role: SYSDBA
    # tnsAdmin: /path/to/wallet
    # vault: { oci|azure|hashicorp: {...} }
    # labels: { env: prod }
metrics:
  default: default-metrics.toml
  custom:
    - custom-metrics-example/custom-metrics.toml
  # scrapeInterval: 15s
log:
  destination: /opt/alert.log
  interval: 15s
  # disable: 0
```

Without `--config.file`, the exporter falls back to env vars `DB_USERNAME`, `DB_PASSWORD`, `DB_CONNECT_STRING`, `DB_ROLE`, `TNS_ADMIN` for a single "default" database.

Runtime env extras (handled in `main.go`):

- `FREE_INTERVAL` - periodically calls `debug.FreeOSMemory()`.
- `RESTART_INTERVAL` - periodically `syscall.Exec`s itself.
- `CONFIG_FILE`, `TELEMETRY_PATH`, `DEFAULT_METRICS`, `CUSTOM_METRICS`, `QUERY_TIMEOUT`, `DATABASE_MAXIDLECONNS`, `DATABASE_MAXOPENCONNS`, `DATABASE_POOLINCREMENT`, `DATABASE_POOLMAXCONNECTIONS`, `DATABASE_POOLMINCONNECTIONS`, `LOG_DESTINATION` - mirror the kingpin flags.

## Testing

```bash
make go-test
# expands to:
GOOS=$(uname -s|tr A-Z a-z) GOARCH=$(go env GOARCH) \
  go test --tags goora -coverprofile=test-coverage.out $(go list ./... | grep -v /vendor/)
```

Tests use the `goora` tag to avoid CGO/Oracle Instant Client. Current unit tests cover `duplicatedLabels` (`collector/collector_test.go`) and `Database.IsValid`/`invalidate` (`collector/database_test.go`). There is no integration test suite checked in beyond the docker-compose harness; `tests/1000-integers.toml` is a sample custom metric used for manual validation.

## Dependencies

- Go 1.24.11 (also pinned in `Dockerfile` via `GO_VERSION`).
- Drivers: `github.com/godror/godror` v0.50 (CGO), `github.com/sijms/go-ora/v2` v2.9 (pure Go).
- Prometheus: `client_golang` v1.23, `common` v0.67, `exporter-toolkit` v0.15.
- Config/CLI: `alecthomas/kingpin/v2`, `BurntSushi/toml`, `gopkg.in/yaml.v2`.
- Vaults: `Azure/azure-sdk-for-go/sdk/security/keyvault/azsecrets`, `oracle/oci-go-sdk/v65`, `hashicorp/vault/api`.
- Runtime container deps (godror image): `oracle-instantclient-basic-23.9.0.25.07`, `glibc 2.28`, base `ghcr.io/oracle/oraclelinux:8-slim`.

The Go module path remains the upstream `github.com/oracle/oracle-db-appdev-monitoring`; only the Docker image coordinates (`IMAGE_NAME=ghcr.io/kubedb/oracle-exporter`) and CI pipeline are KubeDB-specific.

## Code Conventions

- Standard Go formatting; `make go-lint` runs `golangci-lint` v1.50.1 against the repo with config in `.golangci.yml`.
- Logging uses `log/slog` (`*slog.Logger`) constructed from `prometheus/common/promslog`; pass the logger explicitly through constructors (do not use a package-global).
- Driver-specific code is isolated behind `//go:build godror` / `//go:build goora` files exposing the same `connect(logger, dbname, DatabaseConfig) *sql.DB` and `isInvalidCredentialsError(err) bool` symbols. Add new driver-specific branches in those files only.
- Copyright header on every source file: `Copyright (c) <year>, Oracle and/or its affiliates.` followed by the UPL v1.0 SPDX line. Preserve when editing existing files; this project is dual-licensed UPL-1.0 / MIT.
- Public structs in `collector/config.go` use `yaml:"..."` tags. New configuration fields should be added there with matching getters (e.g. `GetQueryTimeout`) when defaults are needed.
- Prometheus metrics use the `oracledb` namespace and per-database `constLabels` derived from `MetricsConfiguration.DatabaseLabel()` plus `DatabaseConfig.Labels`. All databases must expose the union of label keys (blank values when unset) - see the `allConstLabels` loop in `NewExporter`.
- Connection strings are masked via `maskDsn` before logging.
- The KubeDB fork tracks upstream; avoid invasive refactors. Confine fork-specific changes to `Makefile` (image name), `Dockerfile`, and `.github/workflows/` where possible.
