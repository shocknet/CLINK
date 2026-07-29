# CLINK Beacon Specification

## Overview

**CLINK Beacon** is a replaceable Nostr event that a node service publishes to signal **liveness**, **persona** (display metadata), and optional **protocol parameters** to clients that already know the service pubkey and relay.

It complements interactive CLINK protocols (kinds `21001`–`21004`) with a lightweight, subscription-friendly heartbeat clients can watch before opening encrypted round-trips.

## Motivation

Wallet and CLI clients need to know whether a node is reachable before Enroll, payment, or management flows. Polling proprietary HTTP health endpoints reintroduces web infrastructure CLINK avoids.

A periodic beacon on the same relay the service already uses gives clients:

1. **Onlineness** — fresh `created_at` ⇒ service is likely up; stale/missing ⇒ warn or defer work
2. **Persona** — human-readable name, avatar, and related display fields for source lists and UIs
3. **Fee disclosure** — service fee floor and basis points before a pay flow
4. **Enroll fast-path** — optional `enroll_difficulty` so clients can skip an Enroll probe when mining PoW (see [CLINK Enroll](clink-enroll.md))
5. **Capability hints** — optional `supported_kinds` so clients know which CLINK kinds the node advertises
6. **Operator linkage** — optional operator social pubkey so clients can discover or label a node by a known Nostr identity (see **Discovery by operator**). **Requires operator attestation** before trusted display (see **Operator attestation**).

Beacon is **optional for servers** and **optional for clients** as an optimization. Portable behavior always remains available via Enroll probe and normal CLINK request/response flows. Beacon carries **service-level** hints only — not per-account pointers, balances, or wallet RPC (see [CLINK Enroll](clink-enroll.md)).

## Nostr event

Beacon uses [NIP-78](https://github.com/nostr-protocol/nips/blob/master/78.md) **addressable** kind `30078`.

| Field | Value |
|-------|--------|
| **Kind** | `30078` |
| **Author** | Service pubkey (same pubkey as in the service `nprofile` / TLV `0` of CLINK bech32 strings) |
| **`d` tag** | `clink-node` |
| **Tags** | `["operator", "<operator_pubkey_hex>"]` — optional; required when `operator_npub` is present in content (see below) |
| **Content** | UTF-8 JSON object (schema below) |

**Subscription filter (typical):**

```json
{
  "kinds": [30078],
  "authors": ["<service_pubkey_hex>"],
  "#d": ["clink-node"]
}
```

Clients SHOULD subscribe on a relay the service is known to use (from `nprofile`, prior Enroll, or beacon `relays`).

To **discover nodes by operator** (“what does this `npub` run?”), clients SHOULD start from the **operator attestation** (author = operator pubkey), not `#operator` on service beacons. Spoofed services can claim any `operator_npub`; the operator-signed attestation is the authoritative service list.

```json
{
  "kinds": [30078],
  "authors": ["<operator_pubkey_hex>"],
  "#d": ["clink-node-operator"]
}
```

Read `service` tags (and check `clink-node-operator-revoke`). For each attested service pubkey `S`, fetch the `clink-node` beacon from author `S` and confirm it claims the same operator (`operator_npub` / `operator` tag). Only then treat linkage as **verified**.

Clients MAY also query `#operator` on `clink-node` beacons to find **unverified** candidates, but MUST not treat matches as trusted without the operator-first path above.

### Publication cadence

Services that publish beacons SHOULD republish at least every **60 seconds** while online so subscribers can detect staleness without long gaps.

### Client staleness

Clients SHOULD treat a beacon as **stale** when no event with `created_at` within the last **180 seconds** has been seen for that service pubkey and `d` tag. Stale beacons MUST NOT be used for `enroll_difficulty` fast-path; clients fall back to Enroll probe or user warning.

These intervals are recommendations; deployments MAY tune locally but SHOULD stay within similar bounds for interoperable UX.

## Content schema

All fields except `clink_version` are optional. Unknown fields MUST be ignored by clients.

```json
{
  "clink_version": "1",
  "name": "My Node",
  "avatarUrl": "https://example.com/avatar.png",
  "website": "https://example.com",
  "nip05": "bob@example.com",
  "description": "Short human-readable blurb.",
  "operator_npub": "npub1…",
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
| `clink_version` | string | RECOMMENDED | CLINK beacon schema version. Implementations SHOULD use `"1"`. |
| `name` | string | RECOMMENDED when publishing | Display name for the node service. |
| `avatarUrl` | string | optional | HTTPS URL for an avatar image. |
| `website` | string | optional | Service website URL. |
| `nip05` | string | optional | NIP-05 identifier for the service (verification is out of band). |
| `description` | string | optional | Short description for UIs. |
| `operator_npub` | string | optional | Node **operator** social Nostr identity ([NIP-19](https://github.com/nostr-protocol/nips/blob/master/19.md) `npub…` bech32). Not the service CLINK pubkey (event author). When present, the event MUST include tag `["operator", "<hex_pubkey>"]` where hex is the decoded `operator_npub`. Clients MUST NOT treat this as verified without **operator attestation** (below). |
| `relays` | string[] | optional | Relay URL(s) where the service listens for CLINK traffic. First entry MAY be treated as preferred. |
| `fees` | object | optional | Service fee disclosure for pay flows. |
| `fees.serviceFeeFloor` | integer | optional | Minimum service fee in **satoshis**. |
| `fees.serviceFeeBps` | integer | optional | Service fee in basis points (100 = 1%). |
| `enroll_difficulty` | integer | optional | NIP-13 bits the service requires for **new** Enroll when PoW is enabled. MUST **equal** the enforced required difficulty (same value as `required_difficulty` on Enroll code `4`). Not a minimum or maximum — an accurate advertisement. |
| `supported_kinds` | integer[] | optional | CLINK event kinds this service supports (e.g. `21001` Offers, `21002` Debits, `21003` Manage, `21004` Enroll). |

Monetary amounts use **satoshis**, consistent with other CLINK specs.

### Discovery by operator

The **service pubkey** (beacon event author) is the CLINK backend identity. The **operator pubkey** is the human operator’s everyday Nostr key.

**Recommended flow** (operator-first):

1. Fetch current `clink-node-operator` from the operator pubkey → attested service pubkeys (`service` tags).
2. Subtract any pubkeys on current `clink-node-operator-revoke`.
3. For each remaining `S`, fetch `clink-node` from author `S` and confirm `operator_npub` / `operator` tag matches the operator.

Verified “operated by” UI requires steps 1–3. Starting from `#operator` on service beacons invites affinity scams — many spoofed nodes can claim the same famous `npub`.

Operator linkage does **not** grant [Enroll](clink-enroll.md) owner policy on kind `21002` / `21003` for that key unless that key is also the enrolled account owner on the service.

### Operator attestation

A service beacon’s `operator_npub` is a **one-way claim**. Anyone can set `operator_npub` to a famous key. Clients need a **bidirectional proof**: the claimed operator key must also attest that it operates this service pubkey.

The operator publishes a separate replaceable event:

| Field | Value |
|-------|--------|
| **Kind** | `30078` |
| **Author** | Operator pubkey (decoded `operator_npub`) |
| **`d` tag** | `clink-node-operator` |
| **Tags** | `["service", "<service_pubkey_hex>"]` — one tag per attested service (the list lives in tags only) |
| **Content** | `{"clink_version": "1"}` |

Service pubkeys are carried **only** in `service` tags (enables `#service` relay filters). Clients MUST NOT duplicate them in `content`.

**Subscription filter (does operator attest to service `S`?):**

```json
{
  "kinds": [30078],
  "authors": ["<operator_pubkey_hex>"],
  "#d": ["clink-node-operator"],
  "#service": ["<service_pubkey_hex>"]
}
```

**Verification** (operator pubkey `O`, service pubkey `S`):

1. Service beacon (author `S`) includes `operator_npub` / `operator` tag for `O`.
2. Current operator attestation (`d=clink-node-operator`, author `O`) includes tag `["service", "S"]`.
3. Current revocation replaceable state does not include tag `["service", "S"]` — see **Operator attestation revocation**.
4. Steps 1–3 satisfied ⇒ **verified** operator linkage.
5. Beacon claims `O` but attestation missing, no `service` tag for `S`, or `S` is revoked ⇒ **unverified** — clients MUST NOT present as “operated by” without a warning; treat as possible affinity scam.
6. Attestation tags `S` but beacon does not claim `O` ⇒ operator attestation only; no “operated by” UI from beacon alone.

Clients MUST determine attestation and revocation from **current replaceable state** (latest event per `d` tag). Clients MUST NOT order these documents by `created_at` — timestamps are self-reported and not a reliable chain.

**Subscription filter (full attestation document):**

```json
{
  "kinds": [30078],
  "authors": ["<operator_pubkey_hex>"],
  "#d": ["clink-node-operator"]
}
```

### Operator attestation revocation

An operator MAY revoke a service without editing the attestation list by publishing a separate **replaceable** revocation document:

| Field | Value |
|-------|--------|
| **Kind** | `30078` |
| **Author** | Operator pubkey |
| **`d` tag** | `clink-node-operator-revoke` |
| **Tags** | `["service", "<service_pubkey_hex>"]` — one tag per revoked service |
| **Content** | `{"clink_version": "1"}` |

**Subscription filter (revocation document):**

```json
{
  "kinds": [30078],
  "authors": ["<operator_pubkey_hex>"],
  "#d": ["clink-node-operator-revoke"]
}
```

**Subscription filter (is service `S` revoked?):**

```json
{
  "kinds": [30078],
  "authors": ["<operator_pubkey_hex>"],
  "#d": ["clink-node-operator-revoke"],
  "#service": ["<service_pubkey_hex>"]
}
```

**Verification with revocation** (operator `O`, service `S`): fetch the **current** `clink-node-operator` and `clink-node-operator-revoke` replaceable events from `O`. Linkage is **verified** only if attestation includes `["service", "S"]` and revocation does **not**. If both include a `service` tag for `S`, **revocation wins**. To attest again after revoke, the operator MUST republish revocation without the `S` tag (and SHOULD keep attestation consistent).

Clients SHOULD fetch or subscribe to both `clink-node-operator` and `clink-node-operator-revoke` when verifying operator linkage for display.

## Server requirements

Services MAY publish a CLINK beacon. If they publish:

- The event MUST match kind `30078`, author pubkey, and `d` tag `clink-node`.
- `clink_version` SHOULD be `"1"`.
- If `enroll_difficulty` is present, it MUST **equal** the required PoW difficulty enforced on kind `21004` for new enrolls (same value as `required_difficulty` when code is `4`). Services MUST NOT publish a lower or higher value than they enforce.
- If `supported_kinds` is present, listed kinds MUST be actually supported.
- If `fees` is present, values MUST reflect current service fee policy.
- If `operator_npub` is present, the event MUST include `["operator", "<hex_pubkey>"]` and hex MUST match the decoded `operator_npub`. Verified display of operator identity requires matching **operator attestation** from that pubkey.

Services MUST NOT rely on beacon alone for security decisions on kind `21004`; Enroll PoW validation remains on the Enroll request event.

## Client requirements

Clients MAY subscribe to CLINK beacons when they know service pubkey and relay.

When using a beacon:

1. **Onlineness** — use event `created_at` (and subscription freshness) per **Client staleness** above.
2. **Enroll difficulty** — if `enroll_difficulty` is present and the beacon is not stale, clients MAY mine at that value before Enroll instead of probing. The Enroll request still MUST meet the service’s required difficulty (mining **at or above** required satisfies NIP-13; see [CLINK Enroll](clink-enroll.md)). On code `4`, mine at `required_difficulty` and retry once. If absent, stale, or untrusted, clients MUST use the portable Enroll probe path.
3. **Persona / fees** — clients MAY display `name`, `avatarUrl`, etc., and show `fees` before payment; display is advisory unless cross-checked in a pay response.
4. **Relays** — clients MAY update preferred relay hints from `relays` when reconnecting or building filters.
5. **Operator** — for verified linkage, follow **Discovery by operator** (attestation first, then confirm `operator_npub` on each service beacon). Unverified `#operator` beacon matches alone MUST NOT show trusted “operated by” UI.

Clients MUST support Enroll probe (code `4` + `required_difficulty`) regardless of beacon support.

## Relationship to Enroll

```
nprofile (service pubkey + relay)
        │
        ▼
 optional: subscribe kind 30078, d=clink-node
   • stale/missing → treat as unknown / offline warning
   • fresh + enroll_difficulty → MAY skip probe and mine
        │
        ▼
 portable fallback: Enroll probe (kind 21004, 0 PoW → code 4)
        │
        ▼
 kind 21004 Enroll → noffer / ndebit / nmanage
```

Beacon does not create accounts or return pointers. It only advertises parameters and liveness before Enroll.

See [CLINK Enroll](clink-enroll.md) for PoW rules, probe semantics, and pointer delivery.

## Security considerations

- The beacon is a **signed** Nostr event from the service pubkey. The signature proves the service published this persona and hints — not that the service is trustworthy, that quoted `fees` apply to a specific invoice, or that any party is authorized to spend. Clients MUST NOT treat the beacon alone as proof of good behavior, invoice-level fee correctness, or spend authorization.
- **Affinity scam:** a malicious service can publish any `operator_npub`. Clients MUST use bidirectional **operator attestation** and check **revocation** before trusted “operated by” display.
- `nip05` in beacon is not verified by this spec; use NIP-05 lookup when verification matters.
- `enroll_difficulty` fast-path is safe only when the beacon is fresh and from the expected author pubkey; otherwise probe.
- Rate of beacon publication is a operational choice; very sparse beacons weaken onlineness signals.

## Reference implementation

- Spec repo: this document
- **SDK:** [CLINK SDK](https://github.com/shocknet/ClinkSDK) ([`@shocknet/clink-sdk`](https://www.npmjs.com/package/@shocknet/clink-sdk) on npm)
- Reference server: [Lightning.Pub](https://github.com/shocknet/Lightning.Pub) (publishes kind `30078` beacons today; migration to `d=clink-node` and `enroll_difficulty` is pending)
- Reference wallet: [ShockWallet](https://shockwallet.app) (subscribes to service beacons for onlineness and display name)

**Backward compatibility (non-normative):** some deployments historically used `d=Lightning.Pub` with a similar JSON shape. New implementations SHOULD use `d=clink-node`. The `clink-*` prefix reserves the CLINK namespace on kind `30078` (`clink-node`, `clink-node-operator`, `clink-node-operator-revoke`, …). Clients MAY accept legacy `d` tags only when explicitly targeting those deployments during a transition period.
