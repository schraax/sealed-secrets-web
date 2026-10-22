# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**sealed-secrets-web** is a Go web application providing a browser-based UI for [Bitnami Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets). It runs inside a Kubernetes cluster and exposes an API + Vue.js frontend for encoding/decoding, sealing, validating, and viewing Kubernetes secrets without requiring kubectl access.

## Commands

```sh
# Run all tests (generates mocks, tidies deps, lints Helm chart, then runs coverage)
make test

# Run only Go tests with coverage
make test-cover

# Lint Go code (with auto-fix)
make lint

# Lint Go code (CI mode, no auto-fix)
make lint-ci

# Regenerate mocks after interface changes
make mocks

# Tidy Go module dependencies
make tidy

# Build Docker image (uses podman, amd64)
make build

# Lint Helm chart
make helm-lint

# Run end-to-end tests (requires Kind cluster)
make helm-chainsaw

# Local development cluster setup (Kind + sealed-secrets + app)
./run_local.sh
```

Run a single Ginkgo test suite: `go run github.com/onsi/ginkgo/v2/ginkgo ./pkg/handler/...`

## Architecture

The application is a standard Go HTTP server (Gin) that proxies and wraps the Sealed Secrets controller API:

- **`main.go`** — Wires everything together: creates Kubernetes clients, builds the `Handler`, embeds static files and templates via `go:embed`, and starts Gin on port 8080.
- **`pkg/config/`** — CLI flag parsing and config file support. `types.go` defines all application flags. `filter.go` handles field exclusion/inclusion when loading secrets.
- **`pkg/handler/`** — All HTTP handlers registered on the Gin router:
  - `index.go` — Serves the Vue.js SPA, health check, and version endpoint; holds the central `Handler` struct.
  - `kubeseal.go` — `POST /api/kubeseal` seals a full Kubernetes secret (YAML or JSON output via content negotiation).
  - `raw.go` — `POST /api/raw` seals a single value with a specified scope.
  - `dencode.go` — `POST /api/dencode` base64-encodes or decodes secret data fields.
  - `validate.go` — `POST /api/validate` validates a sealed secret against the controller.
  - `certificate.go` — `GET /api/certificate` returns the controller's public certificate.
  - `secrets.go` — `GET /api/secrets` and `GET /api/secret/:namespace/:name` list/load existing sealed secrets from the cluster.
- **`pkg/seal/`** — `Sealer` interface + `APISealer` implementation that wraps the bitnami sealed-secrets Go library. The interface is mocked for tests.
- **`pkg/matcher/`** — Namespace include/exclude filtering (supports plain strings and regex).
- **`pkg/mocks/`** — Generated mocks (via mockgen) for `CoreV1Interface`, `SecretInterface`, `BitnamiV1alpha1Interface`, `SealedSecretInterface`, and `Sealer`. Regenerate with `make mocks`.

### Frontend

`templates/index.html` is a single-file Vue.js + Vuetify app rendered server-side with a `WebContext` struct injected via Go templates. All vendored JS/CSS lives in `static/`.

### Testing

Tests use Ginkgo v2 (BDD) + Gomega. Test files live alongside the code they test (`_test.go`). Shared test utilities are in `pkg/test/`. The `mocks/` packages are used to isolate handler tests from Kubernetes and the sealed-secrets controller.

### Helm Chart

`chart/` contains the Helm chart for cluster deployment. `make helm-lint` validates it; `make helm-docs` regenerates `chart/README.md` from values. E2E tests use the Chainsaw framework via `make helm-chainsaw`.

## Key Conventions

- Content negotiation (`Accept: application/yaml` vs `application/json`) controls `/api/kubeseal` output format.
- The `--disable-load-secrets` flag removes the secrets listing feature entirely; the UI adapts based on the `WebContext` passed from server to template.
- Namespace filtering (`--include-namespaces`, `--exclude-namespaces`, `--use-regex`) is applied when listing secrets.
- Linting is configured in `.golangci.yaml`; the toolbox binary runner is in `.toolbox.mk`.
