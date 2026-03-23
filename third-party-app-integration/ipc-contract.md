# Third-Party App Integration — IPC Contract (uBus/NBAPI-like)

## Purpose and scope

This document defines a proposed IPC contract that a third-party local gateway app uses to onboard and manage DNS enforcement policy on a prpl-based gateway. The style is intentionally aligned with uBus/NBAPI-like interaction patterns and with the response/error naming patterns evidenced in the prpl SSI API JSON specifications.

This contract is documentation-only and is intended to be implemented later.

## Design principles

The contract is designed for idempotency, debuggability, and safe retries. Policy updates must be safe to repeat. Every method returns a structured response with a standard header and, when applicable, a body. Error conditions are surfaced explicitly and should include a human-readable description.

The contract supports eventing. Clients should be able to subscribe to policy/enforcement change events so they can reconcile state rather than polling.

## Service identity and naming

### IPC object name

The service is exposed as a single IPC object:

`Services.Local.Policy`

This name is chosen to align with the `Services.Local.*` naming seen in the SSI API repository. If a different namespace is required, it should still remain under `Services.Local.*` to preserve consistent discovery.

### Versioning

The service must provide a capabilities call that includes a `ContractVersion` field. Breaking changes should increment the major version.

## Common schemas

### Response envelope

All method responses use the following envelope shape, following the SSI API style where a response has a `Header` containing `Name` and optionally `Description`, and a `Body` for data.

#### Response.Header.Name

The response code names are aligned to those in the SSI API docs for DNS proxy domains and hosts, including:

`OK`, `OPERATION_ALREADY_IN_PROGRESS`, `ARGUMENT_NOT_FOUND`, `ARGUMENT_DATA_TYPE_MISMATCH`, `ARGUMENT_REQUIRED_MISSING`, `ARGUMENT_VALUE_NOT_SUPPORTED`, `METHOD_NOT_FOUND`, `OBJECT_NOT_FOUND`, `OPERATION_ILLEGAL`, `OPERATION_ERROR`, `OPERATION_PERMISSION_DENIED`, `OPERATION_TIMEOUT`.

#### Response.Header.Description

`Description` is required for error responses and should be present for `OK` only when it adds useful diagnostics.

### Request metadata

All requests should support an optional metadata object:

`Meta`:
- `RequestId` (string): A caller-provided identifier used for correlation and audit.
- `TimeoutMs` (integer): Upper bound for the request processing time.
- `DryRun` (boolean): If true, validate and compute effective changes but do not apply to enforcement.

If the underlying IPC bus cannot directly carry nested objects, the same content can be represented as a flattened key space; this document describes the logical schema, not the bus encoding.

## Methods

### Services.Local.Policy.GetCapabilities

Returns supported features and contract version.

#### Request

Empty object or `{ "Meta": { ... } }`.

#### Response (OK)

`Body` fields:
- `ContractVersion` (string)
- `SupportedScopes` (list of strings): for example `["DNS"]`
- `DnsEnforcement` (object):
  - `Provider` (string): `"dnsmasq"`
  - `SupportsPerClientRules` (boolean)
  - `SupportsDomainBlocklist` (boolean)
  - `SupportsDomainAllowlist` (boolean)
  - `SupportsRewrite` (boolean)
  - `ReloadStrategy` (string): `"reload"` or `"restart"` or `"signal"`
- `Limitations` (list of strings): includes DoH/DoT limitations.

#### Response errors

`OPERATION_PERMISSION_DENIED` if the caller cannot query capabilities (platform policy choice).

### Services.Local.Policy.Onboarding.Request

Requests an onboarding grant for a specific scope.

#### Request

- `App` (object, required):
  - `Name` (string, required)
  - `Id` (string, required): stable identifier (package id, signing subject, or similar)
  - `Version` (string, optional)
- `RequestedScopes` (list of strings, required): for example `["DNS"]`

#### Response (OK)

`Body` fields:
- `GrantId` (string): identifier for later operations and audit linkage
- `GrantedScopes` (list of strings)
- `ExpiresAt` (string): RFC3339 timestamp, if grants are time-limited

#### Response errors

- `ARGUMENT_REQUIRED_MISSING` if any required fields are missing.
- `ARGUMENT_DATA_TYPE_MISMATCH` if types do not match.
- `OPERATION_PERMISSION_DENIED` if onboarding is disallowed by platform policy.
- `ARGUMENT_VALUE_NOT_SUPPORTED` if requested scopes are not supported.

### Services.Local.Policy.Onboarding.Revoke

Revokes an existing onboarding grant.

#### Request

- `GrantId` (string, required)

#### Response (OK)

Empty `Body`.

#### Response errors

- `ARGUMENT_REQUIRED_MISSING`
- `OPERATION_PERMISSION_DENIED`
- `OPERATION_ERROR` if the revoke cannot be persisted

### Services.Local.Policy.DNS.Policy.Upsert

Creates or updates a DNS policy. This is the primary policy push method.

#### Idempotency

The caller must supply `PolicyId` and `ClientPolicyVersion`. The service should reject stale updates if `ClientPolicyVersion` is lower than the last accepted one, or accept them idempotently if the policy content hash matches.

#### Request

- `GrantId` (string, required)
- `PolicyId` (string, required)
- `ClientPolicyVersion` (integer, required)
- `Policy` (object, required):
  - `Enabled` (boolean, required)
  - `Description` (string, optional)
  - `Rules` (list, required)
    - Each rule is an object with:
      - `RuleId` (string, required)
      - `Action` (string, required): `"ALLOW"`, `"BLOCK"`, `"REWRITE"`
      - `Domain` (string, required): domain name or pattern, constrained by implementation
      - `RewriteTo` (string, optional): only for `"REWRITE"`
      - `ClientSelector` (object, optional):
        - `Mac` (string, optional)
        - `Ip` (string, optional)
        - `Hostname` (string, optional)

#### Response (OK)

`Body` fields:
- `PolicyId` (string)
- `AcceptedPolicyVersion` (integer): service-side accepted version
- `ActivePolicyVersion` (integer): version currently enforced (may lag if apply is async)
- `ApplyStatus` (string): `"APPLIED"`, `"PENDING"`, `"FAILED"`
- `Diagnostics` (object):
  - `AcceptedRules` (list of `RuleId`)
  - `RejectedRules` (list of objects):
    - `RuleId` (string)
    - `Reason` (string)
  - `DowngradedRules` (list of objects):
    - `RuleId` (string)
    - `Reason` (string)
  - `Limitations` (list of strings): can include DoH/DoT notes

#### Response errors

- `OPERATION_PERMISSION_DENIED` if `GrantId` does not authorize DNS policy changes
- `ARGUMENT_VALUE_NOT_SUPPORTED` if rule action or domain encoding is not supported
- `ARGUMENT_DATA_TYPE_MISMATCH`, `ARGUMENT_REQUIRED_MISSING`
- `OPERATION_ALREADY_IN_PROGRESS` if an apply is currently locked
- `OPERATION_ERROR` on internal failures
- `OPERATION_TIMEOUT` if enforcement apply exceeds time budget
- `OPERATION_ILLEGAL` if called in invalid state (for example, onboarding not complete)

### Services.Local.Policy.DNS.Policy.Get

Returns the currently persisted policy and enforcement state.

#### Request

- `GrantId` (string, required)
- `PolicyId` (string, required)

#### Response (OK)

`Body` fields:
- `PolicyId` (string)
- `PersistedPolicyVersion` (integer)
- `ActivePolicyVersion` (integer)
- `Policy` (object): same structure as upsert
- `Enforcement` (object):
  - `Provider` (string): `"dnsmasq"`
  - `State` (string): `"OK"`, `"DEGRADED"`, `"FAILED"`, `"UNKNOWN"`
  - `LastApplyTime` (string): RFC3339 timestamp
  - `LastError` (string, optional)

### Services.Local.Policy.DNS.Policy.Delete

Deletes a policy by id and removes enforcement artifacts.

#### Request

- `GrantId` (string, required)
- `PolicyId` (string, required)

#### Response (OK)

Empty `Body`.

## Events

The IPC bus should publish events so third-party apps can reconcile state.

### Services.Local.Policy.Events.PolicyChanged

Emitted when the persisted policy changes.

Event payload fields:
- `PolicyId` (string)
- `PersistedPolicyVersion` (integer)
- `ChangedAt` (string, RFC3339)
- `Source` (string): `"third-party-app"` or `"system"`

### Services.Local.Policy.Events.EnforcementStateChanged

Emitted when enforcement transitions state (for example dnsmasq reload failure).

Event payload fields:
- `PolicyId` (string)
- `ActivePolicyVersion` (integer)
- `Provider` (string): `"dnsmasq"`
- `State` (string)
- `ChangedAt` (string, RFC3339)
- `Error` (string, optional)

## Authorization and identity binding

Authorization is modeled using `GrantId`. The platform must bind `GrantId` to the app identity and enforce scope checks on each method. The contract does not mandate how identity is proven on the bus; that is a platform decision, but the service must be able to produce auditable records that tie changes to an identity.

## Notes on alignment with existing specs

The error code names and the response envelope are designed to mirror patterns in the SSI API JSON for DNS Proxy Domains and Hosts. The method naming uses a similar fully qualified dot-separated naming convention consistent with that repository’s `operationId` style.

## Out of scope

This contract does not define a complete policy engine. It defines only the boundary contract and minimal DNS enforcement policy elements needed for onboarding and demonstration.

This contract does not define a remote HTTP transport. It assumes local IPC.
