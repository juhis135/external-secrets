# Research: SAP Credential Store Provider

**Feature**: 001-sap-credstore-provider
**Date**: 2026-05-15
**Status**: Complete — all unknowns resolved

---

## Provider Interface Contract

**Decision**: Implement both `esv1.Provider` and `esv1.SecretsClient` on a single
`Provider` struct, consistent with the Infisical pattern. Compliance enforced with
compile-time assertions:

```go
var _ esv1.SecretsClient = &Client{}
var _ esv1.Provider     = &Provider{}
```

**Rationale**: Infisical and Doppler both use this two-struct pattern where `Provider`
handles `NewClient`/`ValidateStore`/`Capabilities` and a separate `Client` handles
the secret operations. Separating auth (provider) from data ops (client) keeps the
reconcile hot-path free of auth setup.

**Alternatives considered**: Single struct implementing both interfaces — rejected
because it conflates lifecycle concerns and makes mocking harder in tests.

---

## Module Structure

**Decision**: New isolated Go module at
`github.com/external-secrets/external-secrets/providers/v1/sapcredentialstore`
under `providers/v1/sapcredentialstore/`.

**Rationale**: Every provider in the repo has its own go.mod with local `replace`
directives pointing at `../../../apis` and `../../../runtime`. This isolates
provider dependencies and keeps the main module's dependency graph clean.

**Go version**: `go 1.26.3` (matching doppler/infisical go.mod).

**Key dependencies**:
- `github.com/external-secrets/external-secrets/apis` (replace → `../../../apis`)
- `github.com/external-secrets/external-secrets/runtime` (replace → `../../../runtime`)
- `sigs.k8s.io/controller-runtime`
- `k8s.io/api`, `k8s.io/client-go` (for Kubernetes types)
- Standard library `net/http`, `encoding/json`, `crypto/tls`, `crypto/x509`

No third-party SAP SDK exists that matches the credential store REST API; the HTTP
client is built from stdlib (`net/http`). This avoids adding a new module dependency
and keeps the CVE surface minimal.

**Alternatives considered**: `github.com/SAP/cloud-security-client-go` for OAuth2 —
rejected because it adds an unvetted dependency for functionality achievable with
`golang.org/x/oauth2` (already in the repo's dependency tree).

---

## Authentication Architecture

### OAuth2 Client Credentials

**Decision**: Use `golang.org/x/oauth2/clientcredentials` package, which:
- Handles token acquisition and automatic refresh
- Is already used elsewhere in the ESO dependency tree
- Produces an `http.Client` with a token source, usable directly for API requests

**Token lifecycle**: Token is acquired on `NewClient`. The `oauth2` transport
refreshes tokens automatically when they expire. The token is in-memory only
and discarded when the controller pod restarts.

**Rationale**: No custom token cache or refresh loop needed — `golang.org/x/oauth2`
handles this correctly out of the box.

### mTLS

**Decision**: Build a custom `*http.Client` with a `tls.Config` that loads the
client certificate and private key from resolved Kubernetes Secret data. Certificate
is parsed on `NewClient` and held in the `tls.Config` for the lifetime of the client.

**Rationale**: `crypto/tls` stdlib is sufficient for mTLS. No external dependency.

### Mutual exclusion of auth modes

**Decision**: `ValidateStore` returns an error if both `oauth2` and `mtls` fields are
non-nil, or if neither is set. This is validated at admission time (webhook) and also
at `NewClient` to guard against race conditions.

---

## CRD API Field Naming

**Decision**:
- Provider field on `SecretStoreProvider`: `sapCredentialStore` (camelCase JSON tag)
- Provider name in the registry: derived automatically as `"sapCredentialStore"`

**Rationale**: Multi-word providers in this repo use camelCase JSON tags
(e.g., `onepassword`, `beyondtrust`). `sapCredentialStore` is clear, unambiguous,
and follows the same convention.

**Note**: The `SecretStoreProvider` struct has `+kubebuilder:validation:MaxProperties=1`,
so only one provider field may be set per `SecretStore`.

---

## Remote Reference Key Format

**Decision**: `ExternalSecretDataRemoteRef` fields are used as follows:
- `ref.Key` → credential name (required)
- `ref.Property` → credential type: `password` (default), `key`, or `certificate`
- `ref.Version` → ignored (SAP Credential Store does not have versioning)

For credentials of type `certificate` that expose multiple fields (`value` = cert
PEM, `key` = private key PEM), a composite property selector is used:
- `ref.Property = "certificate"` → returns certificate PEM (`value` field)
- `ref.Property = "certificate/key"` → returns private key PEM (`key` field)

**Rationale**: The `property` field is the established ESO convention for
sub-field access within a secret (used by AWS, GCP, etc.). Using `/` as a
separator for certificate sub-fields is consistent with how AWS Secrets Manager
handles JSON path access.

**GetSecretMap behavior**: Returns all non-metadata fields of the credential as a
`map[string][]byte`, keyed by field name. For a password: `{"value": ..., "username": ...}`.

**GetAllSecrets behavior**: Lists all credentials across all types (`password`, `key`,
`certificate`) in the configured namespace. Returns them keyed as `<type>/<name>`
to avoid collisions between credential types with identical names.

---

## SAP Credential Store REST API Endpoints

Based on the [SAP Credential Store REST API documentation](https://help.sap.com/docs/credential-store):

| Operation | Method | Path |
|-----------|--------|------|
| Get credential | GET | `{serviceURL}/api/v1/namespaces/{namespace}/credentials/{type}/{name}` |
| List credentials | GET | `{serviceURL}/api/v1/namespaces/{namespace}/credentials?type={type}` |
| Create/Update credential | PUT | `{serviceURL}/api/v1/namespaces/{namespace}/credentials/{type}/{name}` |
| Delete credential | DELETE | `{serviceURL}/api/v1/namespaces/{namespace}/credentials/{type}/{name}` |
| Check existence | HEAD | `{serviceURL}/api/v1/namespaces/{namespace}/credentials/{type}/{name}` |

Credential types in path: `password`, `key`, `certificate`.

**HTTP response shape (password)**:
```json
{
  "name": "db-password",
  "username": "admin",
  "value": "secret-value",
  "metadata": {}
}
```

**HTTP response shape (certificate)**:
```json
{
  "name": "tls-cert",
  "value": "-----BEGIN CERTIFICATE-----...",
  "key": "-----BEGIN PRIVATE KEY-----...",
  "metadata": {}
}
```

---

## Provider Registration

**Decision**: Create `pkg/register/sapcredentialstore.go` with build tag
`//go:build sapcredentialstore || all_providers`.

**Rationale**: Consistent with all other providers in `pkg/register/`. The `all_providers`
tag is used in the default main binary; the named tag allows building a binary with
only this provider.

---

## Test Strategy

| Layer | Approach | Scope |
|-------|----------|-------|
| Unit — provider | Table-driven tests with fake HTTP client interface | ValidateStore, NewClient error paths, auth selection |
| Unit — client | Table-driven tests with fake API interface | GetSecret, GetSecretMap, GetAllSecrets, PushSecret type-mapping |
| Integration | `httptest.NewServer` mocking the SAP REST API | Full request/response cycle for each credential type and auth mode |
| E2E | Future scope — requires live BTP instance | Deferred to post-alpha promotion |

**Rationale**: Integration tests use `httptest` at the HTTP level (not SDK level)
as required by Constitution Principle II and the repo's existing pattern for REST
providers. This catches HTTP encoding/decoding bugs that mock-at-SDK-level tests
would miss.

---

## Performance Profile

**Decision**: No additional caching layer beyond what `golang.org/x/oauth2` provides
for tokens. The provider is stateless between reconcile cycles except for the HTTP
client (which is held by the provider struct for connection reuse).

**Expected cost**:
- 1 OAuth2 token request per token expiry window (typically 1 hour) per `SecretStore`
- 1 REST API call per `ExternalSecret` reconcile
- No unbounded retries — HTTP errors return immediately with a wrapped error; the
  controller's exponential backoff handles re-queuing

**Benchmark target**: Provider adds no measurable overhead to controller startup or
reconcile loop when no SAP `SecretStore` resources are configured.

---

## Security Compliance Summary

| Rule | Implementation |
|------|---------------|
| No credential logging | API responses never logged; error messages include only non-sensitive metadata (credential name, HTTP status code) |
| TLS enforcement | `tls.Config` has no `InsecureSkipVerify`. `http.DefaultTransport` clone used with TLS config set |
| RBAC | No new RBAC rules required — provider reads existing Kubernetes Secrets via the existing `get`/`list` rules on the controller's ClusterRole |
| CVE-clean deps | Uses only stdlib and `golang.org/x/oauth2` (already in repo); no new external modules |
| Credential scope | OAuth2 tokens and mTLS private keys held only in memory in the `*http.Client`; discarded when pod restarts |
