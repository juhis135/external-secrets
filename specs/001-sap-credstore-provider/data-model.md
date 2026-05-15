# Data Model: SAP Credential Store Provider

**Feature**: 001-sap-credstore-provider
**Date**: 2026-05-15

---

## API Types (Go)

**File**: `apis/externalsecrets/v1/secretstore_sapcredentialstore_types.go`

```go
package v1

import esmeta "github.com/external-secrets/external-secrets/apis/meta/v1"

// SAPCredentialStoreProvider configures the SAP Credential Store ESO provider.
type SAPCredentialStoreProvider struct {
    // ServiceURL is the base URL of the SAP Credential Store REST API, as
    // provided in the BTP service binding.
    // Example: https://<instance>.credstore.cfapps.<region>.hana.ondemand.com
    // +kubebuilder:validation:Required
    ServiceURL string `json:"serviceURL"`

    // Namespace is the credential namespace within the SAP Credential Store
    // instance (not a Kubernetes namespace).
    // +kubebuilder:validation:Required
    Namespace string `json:"namespace"`

    // Auth contains authentication credentials for SAP Credential Store.
    // Exactly one of oauth2 or mtls must be specified.
    // +kubebuilder:validation:Required
    Auth SAPCSAuth `json:"auth"`
}

// SAPCSAuth configures authentication for the SAP Credential Store provider.
// Exactly one of OAuth2 or MTLS must be set.
// +kubebuilder:validation:MaxProperties=1
// +kubebuilder:validation:MinProperties=1
type SAPCSAuth struct {
    // OAuth2 configures OAuth2 client credentials (client ID + secret + token URL).
    // Suitable for standard BTP service bindings.
    // +optional
    OAuth2 *SAPCSOAuth2Auth `json:"oauth2,omitempty"`

    // MTLS configures mutual TLS authentication using a client certificate and key.
    // +optional
    MTLS *SAPCSMTLSAuth `json:"mtls,omitempty"`
}

// SAPCSOAuth2Auth holds OAuth2 client credentials for SAP Credential Store.
type SAPCSOAuth2Auth struct {
    // TokenURL is the OAuth2 token endpoint URL.
    // Example: https://<subaccount>.authentication.<region>.hana.ondemand.com/oauth/token
    // +kubebuilder:validation:Required
    TokenURL string `json:"tokenURL"`

    // ClientID is a reference to a Kubernetes Secret key containing the OAuth2
    // client ID.
    // +kubebuilder:validation:Required
    ClientID esmeta.SecretKeySelector `json:"clientId"`

    // ClientSecret is a reference to a Kubernetes Secret key containing the
    // OAuth2 client secret.
    // +kubebuilder:validation:Required
    ClientSecret esmeta.SecretKeySelector `json:"clientSecret"`
}

// SAPCSMTLSAuth holds mTLS certificate credentials for SAP Credential Store.
type SAPCSMTLSAuth struct {
    // Certificate is a reference to a Kubernetes Secret key containing the
    // PEM-encoded client certificate.
    // +kubebuilder:validation:Required
    Certificate esmeta.SecretKeySelector `json:"certificate"`

    // PrivateKey is a reference to a Kubernetes Secret key containing the
    // PEM-encoded private key corresponding to Certificate.
    // +kubebuilder:validation:Required
    PrivateKey esmeta.SecretKeySelector `json:"privateKey"`
}
```

**Field to add to `SecretStoreProvider` struct** in
`apis/externalsecrets/v1/secretstore_types.go`:

```go
// SAPCredentialStore configures this store to sync secrets using the
// SAP Credential Store provider.
// +optional
SAPCredentialStore *SAPCredentialStoreProvider `json:"sapCredentialStore,omitempty"`
```

---

## Provider Struct (Go)

**File**: `providers/v1/sapcredentialstore/provider.go`

```go
// Provider implements esv1.Provider for SAP Credential Store.
// Holds no per-reconcile state — acts as a factory for Client instances.
type Provider struct{}

// Compile-time interface assertions
var _ esv1.Provider = &Provider{}
```

**File**: `providers/v1/sapcredentialstore/client.go`

```go
// Client implements esv1.SecretsClient for SAP Credential Store.
// One Client is created per SecretStore reconcile cycle via Provider.NewClient.
type Client struct {
    api       SAPCSClientInterface  // mockable HTTP client interface
    namespace string                // SAP CS namespace from SecretStore spec
}

// Compile-time interface assertion
var _ esv1.SecretsClient = &Client{}
```

---

## HTTP Client Interface (for mockability)

**File**: `providers/v1/sapcredentialstore/api/client.go`

```go
// SAPCSClientInterface is the interface the HTTP client implements.
// Defined in the api sub-package so the fake can implement it without
// importing the parent package.
type SAPCSClientInterface interface {
    GetCredential(ctx context.Context, ns, credType, name string) (*Credential, error)
    ListCredentials(ctx context.Context, ns, credType string) ([]CredentialMeta, error)
    PutCredential(ctx context.Context, ns, credType, name string, body *CredentialBody) error
    DeleteCredential(ctx context.Context, ns, credType, name string) error
    CredentialExists(ctx context.Context, ns, credType, name string) (bool, error)
}
```

---

## HTTP Response Models

**File**: `providers/v1/sapcredentialstore/api/types.go`

```go
// Credential is the full credential payload returned by GET requests.
type Credential struct {
    Name     string            `json:"name"`
    Username string            `json:"username,omitempty"` // password type only
    Value    string            `json:"value"`              // primary secret value
    Key      string            `json:"key,omitempty"`      // certificate type only (private key PEM)
    Metadata map[string]string `json:"metadata,omitempty"`
}

// CredentialMeta is the list item returned by list endpoints.
type CredentialMeta struct {
    Name string `json:"name"`
    Type string `json:"type"`
}

// CredentialBody is the request payload for PUT (create/update) operations.
type CredentialBody struct {
    Value    string            `json:"value"`
    Username string            `json:"username,omitempty"`
    Key      string            `json:"key,omitempty"`
    Metadata map[string]string `json:"metadata,omitempty"`
}
```

---

## Remote Reference Mapping

| `ref.Key` | `ref.Property` | Returns |
|-----------|---------------|---------|
| `"db-pass"` | `""` or `"password"` | `password` credential's `value` field |
| `"api-key"` | `"key"` | `key` credential's `value` field |
| `"tls-cert"` | `"certificate"` | `certificate` credential's `value` (cert PEM) |
| `"tls-cert"` | `"certificate/key"` | `certificate` credential's `key` field (private key PEM) |

**GetSecretMap** (via `dataFrom.extract`): returns all non-metadata fields of a
single credential as `map[string][]byte`.

**GetAllSecrets** (via `dataFrom.find`): returns all credentials in the namespace,
keyed as `<type>/<name>` to avoid name collisions across credential types.

---

## Credential Type Constants

```go
const (
    credTypePassword    = "password"
    credTypeKey         = "key"
    credTypeCertificate = "certificate"

    defaultCredType = credTypePassword
)
```

---

## Validation Rules

| Field | Rule |
|-------|------|
| `spec.provider.sapCredentialStore.serviceURL` | Required, non-empty |
| `spec.provider.sapCredentialStore.namespace` | Required, non-empty |
| `spec.provider.sapCredentialStore.auth` | Exactly one of `oauth2` or `mtls` must be set |
| `auth.oauth2.tokenURL` | Required when OAuth2 is selected, non-empty |
| `auth.oauth2.clientId.{name,key}` | Required when OAuth2 is selected |
| `auth.oauth2.clientSecret.{name,key}` | Required when OAuth2 is selected |
| `auth.mtls.certificate.{name,key}` | Required when mTLS is selected |
| `auth.mtls.privateKey.{name,key}` | Required when mTLS is selected |

---

## State Transitions

```
SecretStore applied
    │
    ▼
ValidateStore (webhook)
    ├── auth fields missing or ambiguous → Reject (admission error)
    └── valid → Admitted
            │
            ▼
        NewClient (per reconcile)
            ├── resolve auth secrets from Kubernetes
            ├── build http.Client (OAuth2 transport OR mTLS tls.Config)
            └── return Client{api: httpClient, namespace: ns}
                        │
                 ┌──────┴──────────────────────────┐
                 ▼                                  ▼
         GetSecret / GetSecretMap           PushSecret / DeleteSecret
         (ExternalSecret reconcile)         (PushSecret reconcile)
                 │                                  │
                 ▼                                  ▼
         Kubernetes Secret                   SAP Credential Store
         updated/created                     credential created/updated
```
