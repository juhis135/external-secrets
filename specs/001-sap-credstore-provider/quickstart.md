# Quickstart: SAP Credential Store Provider

**Provider**: SAP Credential Store (`sapCredentialStore`)
**Stability**: alpha
**Capabilities**: Read (ExternalSecret), Write (PushSecret), List (dataFrom)

---

## Prerequisites

1. A provisioned SAP Credential Store service instance on SAP BTP.
2. A BTP service binding created for that instance. The binding provides:
   - `serviceURL` — the REST API base URL
   - Either OAuth2 credentials (`clientId`, `clientSecret`, `tokenURL`)  
     or an mTLS certificate + private key
3. The binding credentials loaded into a Kubernetes Secret in your namespace.

---

## Step 1: Create the Kubernetes Secret with binding credentials

**OAuth2 binding**:

```bash
kubectl create secret generic sap-credstore-binding \
  --from-literal=clientId='<your-client-id>' \
  --from-literal=clientSecret='<your-client-secret>' \
  -n my-app
```

**mTLS binding**:

```bash
kubectl create secret generic sap-credstore-binding \
  --from-file=certificate=./client.crt \
  --from-file=privateKey=./client.key \
  -n my-app
```

---

## Step 2: Create the SecretStore

```yaml
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: sap-credstore
  namespace: my-app
spec:
  provider:
    sapCredentialStore:
      serviceURL: "https://<instance>.credstore.cfapps.<region>.hana.ondemand.com"
      namespace: "my-namespace"
      auth:
        oauth2:
          tokenURL: "https://<subaccount>.authentication.<region>.hana.ondemand.com/oauth/token"
          clientId:
            name: sap-credstore-binding
            key: clientId
          clientSecret:
            name: sap-credstore-binding
            key: clientSecret
```

```bash
kubectl apply -f secretstore.yaml
kubectl get secretstore sap-credstore -n my-app
# STATUS should be Valid
```

---

## Step 3: Sync a single credential

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: my-db-password
  namespace: my-app
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: sap-credstore
    kind: SecretStore
  target:
    name: my-db-password-k8s
  data:
    - secretKey: password
      remoteRef:
        key: db-password       # credential name in SAP Credential Store
        property: password     # credential type (default: password)
```

```bash
kubectl apply -f externalsecret.yaml
kubectl get externalsecret my-db-password -n my-app
# STATUS: SecretSynced

kubectl get secret my-db-password-k8s -n my-app -o jsonpath='{.data.password}' | base64 -d
# <your credential value>
```

---

## Step 4: Sync a certificate credential

```yaml
  data:
    - secretKey: tls.crt
      remoteRef:
        key: my-tls-cert
        property: certificate        # certificate PEM

    - secretKey: tls.key
      remoteRef:
        key: my-tls-cert
        property: certificate/key    # private key PEM
```

---

## Step 5: Bulk sync all credentials in a namespace

```yaml
  dataFrom:
    - find:
        name:
          regexp: ".*"
```

The resulting Kubernetes Secret will contain keys like `password/db-pass`,
`key/api-key`, `certificate/tls-cert`.

---

## Step 6: Push a Kubernetes Secret to SAP Credential Store

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
      name: my-api-key-k8s
  data:
    - match:
        secretKey: value
        remoteRef:
          remoteKey: my-api-key
          property: key
```

---

## Validation: All user stories working

```bash
# US1: ExternalSecret synced
kubectl get externalsecret -n my-app
# READY: True

# US2: PushSecret successful
kubectl get pushsecret -n my-app
# STATUS: Synced

# US3: dataFrom bulk sync
kubectl get secret bulk-sync-k8s -n my-app -o json | jq '.data | keys'
# ["certificate/tls-cert", "key/api-key", "password/db-pass"]
```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| `SecretStore` status: Invalid | Missing or conflicting auth fields | Check `ValidateStore` error in events |
| `ExternalSecret` status: Error `401` | OAuth2 credentials invalid or expired | Verify Kubernetes Secret contents |
| `ExternalSecret` status: Error `404` | Credential not found in SAP CS | Check `key` and `property` values match a real credential |
| `ExternalSecret` status: Error `TLS` | mTLS cert/key mismatch or expired | Rotate and re-apply the binding Kubernetes Secret |
