# SecretStore Contract: SAP Credential Store Provider

**API Group**: `external-secrets.io/v1`
**Provider field**: `sapCredentialStore`
**Capabilities**: ReadWrite (ExternalSecret read + PushSecret write)

---

## SecretStore

```yaml
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: sap-credstore
  namespace: my-app
spec:
  provider:
    sapCredentialStore:
      # serviceURL: Base URL from the BTP service binding (required)
      serviceURL: "https://<instance>.credstore.cfapps.<region>.hana.ondemand.com"

      # namespace: The credential namespace within the service instance (required)
      namespace: "my-namespace"

      # auth: Exactly one of oauth2 or mtls must be specified
      auth:
        oauth2:
          tokenURL: "https://<subaccount>.authentication.<region>.hana.ondemand.com/oauth/token"
          clientId:
            name: sap-credstore-binding    # Kubernetes Secret name
            key: clientId                  # key within that Secret
          clientSecret:
            name: sap-credstore-binding
            key: clientSecret
```

**Alternative — mTLS auth**:

```yaml
      auth:
        mtls:
          certificate:
            name: sap-credstore-binding
            key: certificate              # PEM-encoded client certificate
          privateKey:
            name: sap-credstore-binding
            key: privateKey               # PEM-encoded private key
```

**ClusterSecretStore** — same `spec.provider` block, but Secret references require
an explicit `namespace` field in each `SecretKeySelector`.

---

## ExternalSecret — Single Credential

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: db-password
  namespace: my-app
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: sap-credstore
    kind: SecretStore
  target:
    name: db-password-k8s
  data:
    - secretKey: password           # key in the resulting Kubernetes Secret
      remoteRef:
        key: db-password            # credential name in SAP Credential Store
        property: password          # credential type: password (default), key, certificate
```

---

## ExternalSecret — Certificate Credential (PEM cert + private key)

```yaml
  data:
    - secretKey: tls.crt
      remoteRef:
        key: my-tls-cert
        property: certificate        # returns the certificate PEM value

    - secretKey: tls.key
      remoteRef:
        key: my-tls-cert
        property: certificate/key    # returns the private key PEM
```

---

## ExternalSecret — Bulk Sync (dataFrom)

```yaml
  dataFrom:
    - find:
        name:
          regexp: ".*"              # match all credentials in the configured namespace
```

**Resulting Kubernetes Secret keys**: `<type>/<name>`, e.g.:
- `password/db-password`
- `key/api-key`
- `certificate/tls-cert`

---

## PushSecret

```yaml
apiVersion: external-secrets.io/v1alpha1
kind: PushSecret
metadata:
  name: push-api-key
  namespace: my-app
spec:
  refreshInterval: 1h
  secretStoreRefs:
    - name: sap-credstore
      kind: SecretStore
  selector:
    secret:
      name: my-api-key-k8s          # source Kubernetes Secret
  data:
    - match:
        secretKey: value             # key in the source Kubernetes Secret
        remoteRef:
          remoteKey: my-api-key      # credential name to create/update in SAP CS
          property: key              # credential type: password, key, or certificate
```

---

## ValidateStore Error Conditions

| Condition | Error message |
|-----------|---------------|
| `serviceURL` empty | `"sapCredentialStore.serviceURL is required"` |
| `namespace` empty | `"sapCredentialStore.namespace is required"` |
| Neither auth mode set | `"sapCredentialStore.auth: exactly one of oauth2 or mtls must be specified"` |
| Both auth modes set | `"sapCredentialStore.auth: exactly one of oauth2 or mtls must be specified"` |
| `oauth2.tokenURL` empty | `"sapCredentialStore.auth.oauth2.tokenURL is required"` |
| OAuth2 clientId missing | `"sapCredentialStore.auth.oauth2.clientId.name and .key are required"` |
| mTLS certificate missing | `"sapCredentialStore.auth.mtls.certificate.name and .key are required"` |

---

## Status Conditions

The provider uses standard ESO status conditions on `ExternalSecret` and `SecretStore`.

| Condition | Reason | Message example |
|-----------|--------|-----------------|
| `Ready: True` | `SecretSynced` | — |
| `Ready: False` | `SecretSyncedError` | `"failed to get credential 'db-pass' (type: password): 404 Not Found"` |
| `Ready: False` | `InvalidProviderConfig` | `"sapCredentialStore.auth.oauth2.tokenURL is required"` |
| `Ready: False` | `SecretSyncedError` | `"oauth2 token request failed: 401 Unauthorized"` |

---

## Metrics

The provider emits the standard ESO `externalsecrets_provider_api_calls_total` metric via
`metrics.ObserveAPICall("sapCredentialStore", <operation>, err)`.

Operations observed:
- `GetCredential`
- `ListCredentials`
- `PutCredential`
- `DeleteCredential`
- `CredentialExists`
- `OAuth2TokenRequest`
