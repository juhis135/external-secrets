# Feature Specification: SAP Credential Store Provider

**Feature Branch**: `001-sap-credstore-provider`
**Created**: 2026-05-15
**Status**: Draft
**Input**: User description: "Add SAP Credential Store provider" (GitHub issue #6348)

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Sync SAP Credentials into Kubernetes (Priority: P1)

A platform engineer running workloads on SAP Business Technology Platform (BTP) has
credentials stored in the SAP Credential Store service. They want to sync those
credentials into Kubernetes Secrets using a standard `ExternalSecret` manifest so that
their workloads can consume them without any custom tooling.

**Why this priority**: This is the core read path. Without it, SAP BTP users cannot use
ESO at all for this provider. All other stories depend on the provider being able to
authenticate and retrieve credentials.

**Independent Test**: Configure a `SecretStore` pointing at a live SAP Credential Store
instance, create an `ExternalSecret` referencing a specific credential, and verify the
resulting Kubernetes Secret contains the correct value. Fully testable in isolation and
delivers the primary user value.

**Acceptance Scenarios**:

1. **Given** a `SecretStore` is configured with a valid SAP Credential Store service URL,
   namespace, and OAuth2 credentials, **When** an `ExternalSecret` references a password
   credential by name, **Then** the controller creates or updates a Kubernetes Secret
   with the credential's value within the standard reconciliation period.

2. **Given** a `SecretStore` is configured with mTLS credentials, **When** an
   `ExternalSecret` references a certificate credential, **Then** the controller creates
   a Kubernetes Secret containing the certificate material.

3. **Given** a `SecretStore` with invalid or expired authentication credentials,
   **When** the controller attempts to reconcile an `ExternalSecret`, **Then** the
   `ExternalSecret` status reflects a clear, actionable error condition and no Kubernetes
   Secret is created or overwritten with incorrect data.

4. **Given** a referenced credential does not exist in SAP Credential Store, **When**
   the controller reconciles the `ExternalSecret`, **Then** the status condition describes
   a "not found" error rather than a generic failure.

---

### User Story 2 - Push Kubernetes Secrets to SAP Credential Store (Priority: P2)

A platform engineer wants to use ESO's `PushSecret` to write or update a credential in
SAP Credential Store from a Kubernetes Secret, maintaining a single source of truth in
Kubernetes while keeping the external store in sync.

**Why this priority**: Enables the write path for teams whose secret lifecycle originates
in Kubernetes. Important for secret rotation workflows and pairs with the read path to
provide full lifecycle management.

**Independent Test**: Create a Kubernetes Secret, configure a `PushSecret` targeting the
SAP Credential Store provider, and verify the credential appears or is updated in SAP
Credential Store. Testable independently of the read path.

**Acceptance Scenarios**:

1. **Given** a `PushSecret` references a Kubernetes Secret and a target SAP Credential
   Store namespace and credential name, **When** the controller reconciles, **Then** the
   credential is created or updated in SAP Credential Store with the correct value.

2. **Given** a `PushSecret` targets a specific credential type (`password`, `key`, or
   `certificate`), **When** the value is pushed, **Then** the credential is stored under
   the correct type in SAP Credential Store.

3. **Given** the SAP Credential Store rejects the push due to a permissions or
   authentication error, **When** the controller reconciles, **Then** the `PushSecret`
   status reflects the failure with a human-readable message.

---

### User Story 3 - Bulk Sync All Credentials from a Namespace (Priority: P3)

A platform engineer wants to sync all credentials from a SAP Credential Store namespace
into Kubernetes Secrets in a single `ExternalSecret` manifest using `dataFrom`, rather
than listing each credential individually.

**Why this priority**: Reduces operator toil for teams with many credentials in a
namespace. Optional enhancement that builds on the core read path (US1).

**Independent Test**: Configure a `dataFrom` entry on an `ExternalSecret`, verify that
all credentials in the target SAP Credential Store namespace are reflected as keys in
the resulting Kubernetes Secret.

**Acceptance Scenarios**:

1. **Given** an `ExternalSecret` uses `dataFrom` with the SAP Credential Store provider,
   **When** the controller reconciles, **Then** all credentials in the configured
   namespace are enumerated and their values stored as keys in the Kubernetes Secret.

2. **Given** the SAP Credential Store namespace is empty, **When** `dataFrom` is used,
   **Then** the resulting Kubernetes Secret is empty but no error condition is raised.

---

### Edge Cases

- What happens when a SAP Credential Store namespace contains credentials of mixed
  types (`password`, `key`, `certificate`) during a `dataFrom` bulk sync?
- How does the provider behave when the OAuth2 token expires mid-reconciliation?
- What happens if both OAuth2 and mTLS fields are provided in the `SecretStore` spec?
- How is `PushSecret` deletion handled — is the credential removed from SAP
  Credential Store, or left in place?
- What happens when the SAP Credential Store service URL is unreachable or returns
  a non-200 response?

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The system MUST allow users to configure a `SecretStore` or
  `ClusterSecretStore` that authenticates to a SAP Credential Store instance using
  OAuth2 client credentials (client ID, client secret, token URL).
- **FR-002**: The system MUST allow users to configure a `SecretStore` or
  `ClusterSecretStore` that authenticates to a SAP Credential Store instance using
  mTLS (client certificate and key).
- **FR-003**: The system MUST fetch `password`, `key`, and `certificate` credential
  types from SAP Credential Store and expose their values via `ExternalSecret`.
- **FR-004**: The system MUST support writing credentials to SAP Credential Store via
  `PushSecret`, targeting a specific namespace and credential name.
- **FR-005**: The system MUST enumerate all credentials in a SAP Credential Store
  namespace to support `dataFrom` bulk-sync on `ExternalSecret`.
- **FR-006**: The system MUST surface authentication failures, credential-not-found
  conditions, and write errors as `Ready` / `Degraded` status conditions on the
  affected resource with a human-readable `Reason` and `Message`.
- **FR-007**: The system MUST scope credential lookups and writes to a configurable
  SAP Credential Store namespace defined in the `SecretStore` spec.
- **FR-008**: The system MUST reject a `SecretStore` configuration that specifies both
  OAuth2 and mTLS authentication simultaneously.

### Quality & Operational Requirements

- **QR-001**: The provider MUST follow existing ESO repository conventions for
  `SecretStore` provider field naming, status conditions, events, metrics, and package
  layout. No novel patterns without an approved plan exception.
- **QR-002**: The provider MUST include automated tests covering all credential-type
  mappings, both authentication flows, and all defined error conditions. Integration
  tests MUST mock at the HTTP level, not at the SDK level.
- **QR-003**: All user-facing documentation (CRD field descriptions, a provider guide,
  and at least one example `SecretStore` and `ExternalSecret` manifest) MUST be
  delivered in the same change as the implementation.
- **QR-004**: The provider MUST NOT introduce measurable degradation to the existing
  controller reconciliation cycle or memory footprint when zero SAP `SecretStore`
  resources are configured in the cluster.
- **QR-005**: The provider MUST NOT log or expose credential values (OAuth2 secrets,
  mTLS private keys, or fetched credential data) in any log, event, or status field.
  TLS certificate validation MUST NOT be disabled on any SAP Credential Store
  communication path.

### Key Entities

- **SAPCredentialStoreProvider**: The `SecretStore` provider configuration block
  containing the service URL, the target Credential Store namespace, and exactly one
  authentication mode (OAuth2 or mTLS). All values are resolved from Kubernetes Secret
  references.
- **Credential**: A named secret item in SAP Credential Store, typed as `password`,
  `key`, or `certificate`, scoped to a namespace within the service instance.
- **ServiceBinding**: The BTP-issued artifact that provides the service URL and
  authentication material. Values are pre-loaded into a Kubernetes Secret by the
  user before the `SecretStore` is created.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A platform engineer can sync a credential from SAP Credential Store into
  a Kubernetes Secret using only a `SecretStore` and `ExternalSecret` manifest, with
  no custom tooling, init containers, or CronJobs required.
- **SC-002**: A synced credential appears in Kubernetes within the standard ESO
  reconciliation window (≤60 seconds under normal cluster conditions) after the
  `ExternalSecret` is created.
- **SC-003**: A `PushSecret` successfully writes a credential to SAP Credential Store
  within one reconciliation cycle after the source Kubernetes Secret is created or
  updated.
- **SC-004**: `dataFrom` bulk-sync retrieves and reflects all credentials present in
  the configured SAP Credential Store namespace without the user having to enumerate
  individual credential names.
- **SC-005**: Any authentication or retrieval failure produces a status condition with a
  message specific enough for an operator unfamiliar with ESO internals to identify the
  corrective action without consulting source code or logs.

## Assumptions

- Users have a provisioned SAP Credential Store service instance on BTP and hold the
  service binding credentials before creating any `SecretStore`.
- The service binding credentials (client ID, client secret, token URL, or mTLS
  certificate material) are pre-loaded into a Kubernetes Secret before the `SecretStore`
  is applied.
- The provider targets the SAP Credential Store REST API directly; any future
  SDK-based integration is out of scope for this feature.
- `PushSecret` deletion behavior follows the existing ESO `deletionPolicy` convention;
  no custom deletion semantics are introduced.
- The `certificate` credential type supports PEM-encoded data; binary certificate
  formats are out of scope for v1.
- The provider is introduced at `alpha` stability, consistent with the ESO convention
  for new providers. Promotion to `beta` or `stable` is a separate process.
- Only one authentication mode (OAuth2 or mTLS) may be active per `SecretStore`;
  mixed-mode configurations are rejected at admission or reconciliation time.
