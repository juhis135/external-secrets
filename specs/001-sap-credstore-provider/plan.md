# Implementation Plan: SAP Credential Store Provider

**Branch**: `001-sap-credstore-provider` | **Date**: 2026-05-15 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/001-sap-credstore-provider/spec.md`

## Summary

Add a new ESO provider for SAP Credential Store — the native BTP secrets service.
The provider supports reading credentials via `ExternalSecret`, writing via `PushSecret`,
and bulk listing via `dataFrom`. It authenticates using either OAuth2 client credentials
or mTLS, consistent with how BTP service bindings are issued. The implementation follows
the established ESO provider pattern (isolated Go module, `Provider` + `Client` struct
pair, `httptest`-based integration tests) with no new external dependencies beyond
`golang.org/x/oauth2` which is already in the repo's dependency tree.

## Technical Context

**Language/Version**: Go 1.26.3
**Primary Dependencies**: `golang.org/x/oauth2/clientcredentials` (OAuth2), stdlib
`net/http` + `crypto/tls` (mTLS), `sigs.k8s.io/controller-runtime`, ESO apis + runtime
modules
**Storage**: N/A — stateless provider; all persistent state is in Kubernetes Secrets
and SAP Credential Store
**Testing**: `go test`, `net/http/httptest` for HTTP mocking, table-driven unit tests
**Target Platform**: Kubernetes controller (Linux, amd64/arm64)
**Project Type**: Kubernetes operator provider plugin
**Performance Goals**: No overhead when zero SAP `SecretStore` resources are configured.
≤1 external HTTP call per `ExternalSecret` reconcile. OAuth2 token reused until expiry.
**Constraints**: Secret/credential values must never appear in logs or status. TLS
certificate validation must not be bypassed. No new module dependencies that introduce
CVEs.
**Scale/Scope**: Single provider implementation. Targets all clusters using SAP BTP
Credential Store. Alpha stability at launch.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

- **Code quality**: The provider uses the exact two-struct pattern (`Provider` +
  `Client`) established by infisical and doppler. Package layout mirrors
  `providers/v1/infisical/`. Compile-time interface assertions enforce
  `esv1.SecretsClient` and `esv1.Provider` compliance. Error handling follows
  `fmt.Errorf("context: %w", err)` throughout. Documentation surfaces: API types
  file, provider guide in `docs/provider/`, example manifests. ✅ No deviations.

- **Testing**: Unit tests cover all credential type mappings, auth mode selection,
  and `ValidateStore` logic. Integration tests use `httptest.NewServer` at the HTTP
  level (not SDK mock) per Constitution Principle II. Test strategy documented in
  `research.md`. ✅ Tests are mandatory, not optional.

- **Consistency**: Provider field name `sapCredentialStore` (camelCase JSON tag)
  follows multi-word provider convention. Status conditions use standard ESO
  `Ready`/`SecretSynced`/`SecretSyncedError` reasons. Metrics use
  `metrics.ObserveAPICall("sapCredentialStore", ...)`. Module path follows
  `providers/v1/sapcredentialstore` pattern. Registration via
  `pkg/register/sapcredentialstore.go` with `//go:build sapcredentialstore || all_providers`
  build tag. ✅ No undocumented inconsistencies.

- **Performance**: Provider holds HTTP client with connection reuse between reconcile
  cycles. OAuth2 transport handles token refresh — no per-reconcile token requests.
  No unbounded retries (HTTP errors returned immediately; controller handles backoff).
  Zero overhead when no SAP `SecretStore` resources exist. ✅ Documented.

- **Security and compliance**: Credential values never logged (API responses returned
  as `[]byte`, not logged). TLS: no `InsecureSkipVerify`. mTLS private keys held in
  `tls.Config` only; discarded on pod restart. No new RBAC rules required. Only new
  module is `golang.org/x/oauth2` (already a transitive dep in the repo). ✅ Fully
  compliant with Principle V.

## Project Structure

### Documentation (this feature)

```text
specs/001-sap-credstore-provider/
├── plan.md              # This file
├── research.md          # Phase 0 output ✅
├── data-model.md        # Phase 1 output ✅
├── quickstart.md        # Phase 1 output ✅
├── contracts/
│   └── secretstore.md   # Phase 1 output ✅
└── tasks.md             # Phase 2 output (via /speckit.tasks)
```

### Source Code (repository root)

```text
# New files for this feature:

apis/externalsecrets/v1/
├── secretstore_sapcredentialstore_types.go    # NEW — API type definitions
└── secretstore_types.go                       # MODIFY — add sapCredentialStore field

providers/v1/sapcredentialstore/
├── go.mod                                     # NEW
├── go.sum                                     # NEW (generated)
├── provider.go                                # NEW — Provider struct, NewClient, ValidateStore
├── client.go                                  # NEW — Client struct, SecretsClient interface
├── api/
│   ├── client.go                              # NEW — SAPCSClientInterface + httpClient
│   └── types.go                               # NEW — Credential, CredentialMeta, CredentialBody
├── fake/
│   └── fake.go                                # NEW — fake SAPCSClientInterface for tests
├── provider_test.go                           # NEW
└── client_test.go                             # NEW

pkg/register/
└── sapcredentialstore.go                      # NEW — build-tag-gated registration

docs/provider/
└── sap-credential-store.md                    # NEW — user-facing guide

# Codegen-affected files (run `make generate` after API type changes):
apis/externalsecrets/v1/zz_generated.deepcopy.go    # REGENERATED
config/crds/bases/external-secrets.io_secretstores.yaml  # REGENERATED
```

**Structure Decision**: Isolated Go module under `providers/v1/sapcredentialstore/`
with an `api/` sub-package for the HTTP client interface and types. This matches the
doppler layout and keeps the mockable interface in a separate package from the provider
logic that uses it.

## Complexity Tracking

No complexity violations. All patterns follow established repository conventions exactly.
