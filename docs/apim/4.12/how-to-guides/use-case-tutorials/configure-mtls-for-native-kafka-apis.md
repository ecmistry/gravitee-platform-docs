# Configure Mtls For Native Kafka Apis

## Overview

Gravitee now supports mutual TLS (mTLS) as a plan security type for Native Kafka APIs, enabling client certificate-based authentication at the Kafka gateway. This extends the existing mTLS policy — previously HTTP-only — to the Kafka protocol layer, giving API platform administrators a third authentication category alongside Keyless and authentication-based plans (OAuth2, JWT, and API Key) for securing Kafka API access.

## mTLS Authentication on the Kafka Gateway

When a Kafka client connects to the gateway under an mTLS plan, the gateway performs a mutual TLS handshake requiring the client to present a valid certificate. The gateway verifies the client certificate against a configured truststore. The MD5 digest of the first peer certificate is then computed and used as a security token to look up an active subscription. If the certificate is absent, unverifiable, or does not match a subscription, the connection is rejected and the socket is closed cleanly. On success, the connection proceeds normally.

## Plan Security Type Groups and Mutual Exclusivity

For Native Kafka APIs, plan security types are organized into three mutually exclusive groups:

| Group | Plan Types |
|:------|:-----------|
| **Keyless** | Keyless |
| **mTLS** | mTLS |
| **Authentication** | OAuth2, JWT, API Key |

Only one group may have published plans at a time on a given Native Kafka API. Multiple published plans within the same group are allowed. Attempting to publish a plan from one group when a plan from a different group is already published triggers a conflict error and presents a confirmation dialog listing the plans that will be automatically closed.

| Plan Being Published | Conflicts With (already published) |
|:---------------------|:-----------------------------------|
| Keyless | Any mTLS or authentication plan |
| mTLS | Any Keyless or authentication plan |
| Authentication (OAuth2, JWT, API Key) | Any Keyless or mTLS plan |
| Same group as existing plans | No conflict — allowed |

## mTLS Policy Plugin Version Compatibility

The mTLS policy plugin now implements both HTTP and Kafka security interfaces.

| Plugin Version | Supported APIM Versions |
|:---------------|:------------------------|
| `2.x` | APIM 4.10 and later |
| `1.x` | APIM 4.5 through 4.9 |

Upgrading to plugin version `2.x` is required when running APIM 4.10 or later.

## Prerequisites

Before configuring mTLS plans on a Native Kafka API, ensure all of the following conditions are met:

- APIM 4.10 or later with the mTLS policy plugin version `2.x` installed.
- A Native Kafka API (API type `native`, definition version `4.0.0`) with a Kafka listener configured.
- The Kafka gateway is configured with SSL client authentication enabled (`kafka.ssl.clientAuth=required`) and a truststore containing the CA that signed the client certificates.
- Client applications possess a valid client certificate signed by the trusted CA configured in the gateway truststore.
- Any client code or tooling that reads `NativePlanAuthenticationConflictException` parameters has been updated: the parameter `planToPublishIsKeyless` (boolean string) has been replaced by `planToPublishType` (enum name string, e.g., `"KEY_LESS"`, `"MTLS"`, `"API_KEY"`).
- Native API plans retain the single-flow constraint: each plan may have no more than one flow.

## Configuring the Kafka Gateway for mTLS

### SSL/mTLS Settings

The following properties configure the Kafka gateway for mTLS. Truststore properties control which client certificates the gateway accepts; keystore properties configure the gateway's own server certificate.

#### Client Authentication

| Property | Description | Example |
|:---------|:------------|:--------|
| `kafka.ssl.clientAuth` | Enforces mutual TLS by requiring a client certificate. Set to `required` to enable mTLS. | `required` |

#### Truststore (Client Certificate Verification)

| Property | Description | Example |
|:---------|:------------|:--------|
| `kafka.ssl.truststore.type` | Truststore format used to verify client certificates. | `jks` |
| `kafka.ssl.truststore.path` | Path to the JKS truststore file containing the CA that signed client certificates. | `/etc/gravitee/truststore.jks` |
| `kafka.ssl.truststore.password` | Password for the truststore file. | *(secure value)* |

#### Keystore (Gateway Server Certificate)

| Property | Description | Example |
|:---------|:------------|:--------|
| `kafka.ssl.keystore.type` | Keystore format for the gateway's server certificate. | `jks` |
| `kafka.ssl.keystore.path` | Path to the JKS keystore file for the gateway server. | `/etc/gravitee/keystore.jks` |
| `kafka.ssl.keystore.password` | Password for the keystore file. | *(secure value)* |

### Security Chain Timeout

The gateway security chain execution for Kafka connections times out after **20 seconds**. If the timeout elapses before authentication completes, the connection is terminated with a runtime timeout error.

## Creating an mTLS Plan

To add an mTLS plan to a Native Kafka API, navigate to the **Plans** section of your Kafka API in the API Management Console.

1. Click **Add plan** to open the plan creation dialog.
2. Select **mTLS** from the plan type menu.
3. Enter a **Name** for the plan.
4. Complete any remaining plan configuration fields (tags, flows) as needed. No additional security configuration properties are required for the mTLS plan type.
5. Set the plan **Status** to `Published` to make it active.

### mTLS Plan Fields Reference

| Field | Description |
|:------|:------------|
| Name | Display name for the plan. |
| Security type | `mTLS` — certificate-based; no additional configuration parameters. |
| Status | `Published` to activate; must not conflict with existing Keyless or authentication plans. |
| Flows | Optional; maximum one flow per plan on Native APIs. |

### Conflict Dialog Behavior

When publishing an mTLS plan, if a Keyless or authentication plan is already published on the API, the console displays a confirmation dialog: **"Publish plan and close current one"** (or **"Publish plan and close current ones"** for multiple conflicts). The dialog identifies the mTLS plan being published and lists each conflicting plan by name and type that will be automatically closed. A subscription-closure warning is shown when applicable.

### Plan List Type Column

The **Type** column in the plan list table displays human-readable labels for all plan security types:

| Security Type | Display Label |
|:--------------|:-------------|
| `KEY_LESS` | Keyless |
| `MTLS` | mTLS |
| `API_KEY` | API Key |
| `OAUTH2` | OAuth2 |
| `JWT` | JWT |

## Managing Native Kafka API Plans

### Security Step Banner

The informational banner in the plan security step of the Native Kafka API creation wizard reads:

> "Kafka APIs cannot mix Keyless, mTLS, and authentication (OAuth2, JWT, API Key) plans. In order to automatically deploy your API, choose one type: either Keyless, mTLS, or authentication plans."

### Subscription Creation Errors

When subscription creation fails, the error notification displays the specific error message returned by the API. If the API does not return a message, the generic fallback text is shown instead.

### Authentication Conflict Errors

If a plan publish is attempted that violates the mutual-exclusivity rules, the following errors are raised:

| Plan Type Being Published | Error Message |
|:--------------------------|:--------------|
| Keyless | "A plan with mTLS or authentication is already published for the Native API. Keyless plans cannot be combined with mTLS or authentication plans." |
| mTLS | "A Keyless or authentication plan is already published for the Native API. mTLS plans cannot be combined with Keyless or authentication plans." |
| Authentication (API Key, OAuth2, JWT) | "A Keyless or mTLS plan is already published for the Native API. Authentication plans cannot be combined with Keyless or mTLS plans." |

## Gateway Certificate Validation Failure Conditions

If mTLS authentication fails at the gateway, the connection is rejected and the socket is closed. The following conditions cause rejection:

| Condition | Error |
|:----------|:------|
| No TLS session present on the connection | SSL session required |
| Client certificate cannot be verified | Client certificate invalid |
| No client certificate presented | Client certificate missing |
