# CLINK Beacon Specification

## Overview

**CLINK Beacon** is a replaceable Nostr event that a node service publishes to signal **liveness**, **persona** (display metadata), and optional **protocol parameters** to clients that already know the service pubkey and relay.

It complements interactive CLINK protocols (kinds `21001`–`21004`) with a lightweight, subscription-friendly heartbeat clients can watch before opening encrypted round-trips.

## Motivation

A beacon makes it more efficient for clients to learn whether a node is reachable before Enroll, payment, or management flows. Polling proprietary HTTP health endpoints reintroduces web infrastructure that CLINK was created to avoid.

A periodic beacon on the same relay the service already uses gives clients:

1. **Onlineness** — fresh `created_at` ⇒ service is likely up; stale/missing ⇒ warn or defer work
2. **Persona** — human-readable name, avatar, and related display fields for source lists and UIs
3. **Fee disclosure** — service fee floor and basis points before a pay flow
4. **Enroll fast-path** — optional `enroll_difficulty` so clients can skip an Enroll probe when mining PoW (see [CLINK Enroll](clink-enroll.md))
5. **Capability hints** — optional `supported_kinds` so clients know which CLINK kinds the node advertises
6. **Operator linkage** — optional `operator` tag for the operator’s social pubkey. Trusted “operated by” display requires attestation (see **Discovery by operator**).

Beacon is **optional for servers** and **optional for clients** as an optimization. Portable behavior always remains available via Enroll probe and normal CLINK request/response flows. Beacon carries **service-level** hints only — not per-account pointers, balances, or wallet RPC (see [CLINK Enroll](clink-enroll.md)).

## Nostr event

Beacon uses [NIP-78](https://github.com/nostr-protocol/nips/blob/master/78.md) **addressable** kind `30078`.

| Field | Value |
|-------|--------|
| **Kind** | `30078` |
| **Author** | Service pubkey (same pubkey as in the service `nprofile` / TLV `0` of CLINK bech32 strings) |
| **Tags** | Required: `["d", "clink-node"]`, `["clink_version", "1"]`. Optional: `["operator", "<operator_pubkey_hex>"]`. |
| **Content** | UTF-8 JSON object (schema below) |

### Protocol Versioning

CLINK events utilize a mandatory `["clink_version", "1"]` tag. This ensures:
1. **Disambiguation:** Explicitly identifies events belonging to the CLINK protocol, preventing conflicts if other NIPs use the same event kind (`30078`).
2. **Version Compatibility:** Allows clients and services to verify they are using compatible versions of the CLINK protocol specification. Future versions may increment the version number (e.g., `"2"`).

Implementations MUST include this tag in beacon, operator attestation, and revocation events.

**Subscription filter (typical):**

```json
{
  "kinds": [30078],
  "authors": ["<service_pubkey_hex>"],
  "#d": ["clink-node"]
}
```

Clients leveraging beacons MUST subscribe on a relay the service is known to use (from `nprofile`, `noffer`, `ndebit`, `nmanage`, prior Enroll, or beacon `relays`).

For operator discovery and verification, clients SHOULD follow **Discovery by operator**. An `operator` claim on a service beacon alone is unverified because spoofed services can claim any operator.

### Publication cadence

Services that publish beacons SHOULD republish at least every **60 seconds** while online so subscribers can detect staleness without long gaps.

### Client staleness

Clients SHOULD treat a beacon as **stale** when no event with `created_at` within the last **180 seconds** has been seen for that service pubkey and `d` tag. Stale beacons MUST NOT be used for `enroll_difficulty` fast-path; clients fall back to Enroll probe or user warning.

Clients SHOULD treat a beacon whose `created_at` is more than **30 seconds in the future** as invalid for freshness and protocol-parameter decisions.

These intervals are recommendations; deployments MAY tune locally but SHOULD stay within similar bounds for interoperable UX.

## Content schema

All fields are optional. Unknown fields MUST be ignored by clients.

```json
{
  "name": "My Node",
  "avatarUrl": "https://example.com/avatar.png",
  "website": "https://example.com",
  "description": "Short human-readable blurb.",
  "relays": ["wss://relay.example.com"],
  "fees": {
    "serviceFeeFloor": 0,
    "serviceFeeBps": 100
  },
  "enroll_difficulty": 18,
  "supported_kinds": [21001, 21002, 21003, 21004]
}
```

### Field definitions

| Field | Type | Requirement | Description |
|-------|------|-------------|-------------|
| `name` | string | optional | Display name for the node service. |
| `avatarUrl` | string | optional | HTTPS URL for an avatar image. |
| `website` | string | optional | Service website URL. |
| `description` | string | optional | Short description for UIs. |
| `relays` | string[] | optional | Valid WebSocket relay URL(s) where the service listens for CLINK traffic. Production entries SHOULD use `wss:`. First entry MAY be treated as preferred. |
| `fees` | object | optional | Service fee disclosure for pay flows. |
| `fees.serviceFeeFloor` | integer | optional | Minimum service fee in **satoshis**. |
| `fees.serviceFeeBps` | integer | optional | Service fee in basis points (100 = 1%). |
| `enroll_difficulty` | integer | optional | NIP-13 bits the service requires for **new** Enroll when PoW is enabled. MUST **equal** the enforced required difficulty (same value as `required_difficulty` on Enroll code `5`). Not a minimum or maximum — an accurate advertisement. |
| `supported_kinds` | integer[] | optional | CLINK event kinds this service supports (e.g. `21001` Offers, `21002` Debits, `21003` Manage, `21004` Enroll). |

Monetary amounts use **satoshis**, consistent with other CLINK specs.

For a payment amount `amount_sats`, the advertised service fee is the greater of `serviceFeeFloor` and `amount_sats × serviceFeeBps / 10,000`. Services MAY round a fractional result up to the next whole satoshi, since millisatoshis are accounting precision rather than independently settleable value. This remains an advisory disclosure; the payment flow is authoritative.

### Discovery by operator

The **service pubkey** (beacon event author) is the CLINK backend identity. The **operator pubkey** is the human operator’s everyday Nostr key.

**Verification flow** (operator-first):

1. Fetch current `clink-node-operator` from the operator pubkey → attested service pubkeys (`service` tags).
2. Subtract any pubkeys on current `clink-node-operator-revoke` (if both list `S`, **revocation wins**).
3. For each remaining `S`, fetch `clink-node` from author `S` and confirm tag `["operator", "<operator_pubkey_hex>"]`.

Only then may a client show verified “operated by” UI. Trusting the `operator` claim on a service beacon alone invites affinity scams — spoofed nodes can claim any famous `npub`.

### Operator attestation

A service beacon’s `operator` tag is a **one-way claim**. Anyone can tag a famous key. Clients need **bidirectional proof**: the claimed operator key must also attest this service pubkey.

The operator publishes a separate replaceable event:

| Field | Value |
|-------|--------|
| **Kind** | `30078` |
| **Author** | Operator pubkey |
| **Tags** | Required: `["d", "clink-node-operator"]`, `["clink_version", "1"]`. `["service", "<service_pubkey_hex>"]` per attested service. |
| **Content** | `""` |

Service pubkeys are carried **only** in `service` tags. Clients inspect these tags after fetching the document by operator author and `d` tag. Clients MUST NOT duplicate them in `content`.

Clients MUST use the **current** attestation and revocation events per NIP-01 replacement rules. Do **not** compare `created_at` across the two documents; membership in the current revocation document wins.

### Operator attestation revocation

An operator MAY revoke a service without editing the attestation list by publishing a separate **replaceable** revocation document:

| Field | Value |
|-------|--------|
| **Kind** | `30078` |
| **Author** | Operator pubkey |
| **Tags** | Required: `["d", "clink-node-operator-revoke"]`, `["clink_version", "1"]`. `["service", "<service_pubkey_hex>"]` per revoked service. |
| **Content** | `""` |

**Subscription filter (revocation document):**

```json
{
  "kinds": [30078],
  "authors": ["<operator_pubkey_hex>"],
  "#d": ["clink-node-operator-revoke"]
}
```

To re-attest after revoke, the operator MUST republish revocation without the corresponding `service` tag and keep attestation consistent. Before treating operator linkage as verified, clients MUST check both the current attestation and revocation documents.

## Server requirements

Services MAY publish a CLINK beacon. If they publish:

- The event MUST match the **Nostr event** table above (`30078`, service author, required tags).
- Advertised `enroll_difficulty`, `supported_kinds`, and `fees` MUST match what the service actually enforces.
- Services MUST NOT rely on beacon alone for security decisions on kind `21004`; Enroll PoW validation remains on the Enroll request event.

## Client requirements

Clients MAY subscribe to CLINK beacons when they know service pubkey and relay.

Before using a beacon, clients MUST verify its NIP-01 event ID and signature, required kind and tags, and that its author is the expected service pubkey. Clients verifying operator attestation or revocation documents MUST likewise verify their event IDs, signatures, required kind and tags, and expected operator author.

When using a beacon:

1. **Onlineness** — use event `created_at` (and subscription freshness) per **Client staleness** above.
2. **Enroll difficulty** — if `enroll_difficulty` is present and the beacon is valid per **Client staleness** above, clients MAY use it as the Enroll PoW fast-path (see [CLINK Enroll](clink-enroll.md)). Otherwise fall back to the portable Enroll probe path.
3. **Persona / fees** — clients MAY display `name`, `avatarUrl`, etc., and show `fees` before payment; display is advisory unless cross-checked in a pay response.
4. **Relays** — clients MAY update preferred relay hints from `relays` when reconnecting or building filters.
5. **Operator** — clients MUST follow **Discovery by operator** for verified linkage. An unverified `operator` claim on a service beacon alone MUST NOT show trusted “operated by” UI.

## Security considerations

- The beacon is a **signed** event from the service pubkey. That proves authorship of the persona/hints — not trustworthiness, invoice-level fee correctness, or spend authorization.
- **Affinity scam:** a malicious service can tag any operator pubkey. Require bidirectional attestation and check revocation before “operated by” UI.
- `enroll_difficulty` fast-path only when the beacon is valid per **Client staleness** and from the expected author; otherwise probe.

## Reference Implementations

- Spec repo: this document
- **SDK:** [CLINK SDK](https://github.com/shocknet/ClinkSDK) ([`@shocknet/clink-sdk`](https://www.npmjs.com/package/@shocknet/clink-sdk) on npm)
- Reference server: [Lightning.Pub](https://github.com/shocknet/Lightning.Pub) (publishes kind `30078` beacons today; migration to `d=clink-node` and `enroll_difficulty` is pending)
- Reference wallet: [ShockWallet](https://shockwallet.app) (subscribes to service beacons for onlineness and display name)

**Backward compatibility (non-normative):** some deployments historically used `d=Lightning.Pub` with a similar JSON shape. New implementations SHOULD use `d=clink-node`. The `clink-*` prefix reserves the CLINK namespace on kind `30078` (`clink-node`, `clink-node-operator`, `clink-node-operator-revoke`, …). Clients MAY accept legacy `d` tags only when explicitly targeting those deployments during a transition period.
