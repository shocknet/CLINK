# CLINK Enroll Specification

## Overview

**CLINK Enroll** binds a Nostr key to an account on a node service and returns that account’s default static CLINK resource pointers (`noffer1…`, `ndebit1…`, `nmanage1…`).

It is how headless clients, CLIs, and agents that act **as** the user (direct-use principal) send an **Enroll request** to provision an account on a node. It does **not** grant third parties spend or manage rights.

**Terminology:** An **Enroll request** is a kind `21004` event (same request/response pattern as Offers and Debits). Successful enrollment **provisions** an account for the requestor’s signing key. This is not wallet “bootstrap node”, Pub bootstrap liquidity, or application first-run initialization.

## Motivation

Offers, Debits, and Manage use a service pubkey, relay, and resource-specific identifiers. Without Enroll, clients must use proprietary wallet RPC (e.g. Lightning.Pub `GetUserInfo`) to create an account and learn the resulting resource pointers.

Enroll makes account provisioning a portable CLINK protocol so a client can:

1. Talk to a service (`nprofile` / pubkey + relay)
2. Ensure an account exists for its signing key
3. Receive default `noffer` / `ndebit` / `nmanage` strings
4. Proceed with kinds 21001–21003 only

## Non-goals

Enroll MUST NOT:

- Create debit authorizations for other npubs
- Create manage authorizations for other npubs
- Expose balance, history, on-chain, or general wallet RPC
- Replace [CLINK Debits](clink-debits.md) or [CLINK Manage](clink-manage.md)

Third-party spend/manage remains Debits / Manage with explicit grants. Owner self-use is defined under **Owner policy** below (and MAY be implemented as server policy on 21002 / 21003 without a separate Enroll action).

## Target

Input is the **node service** identity only:

- Service pubkey (32-byte)
- Relay URL(s) where the service listens

There is no separate account pointer in an Enroll request: the service pubkey is the request target. The useful resource-specific identifiers are returned inside the resulting `noffer`, `ndebit`, and `nmanage` strings.

## Nostr Events

- **Kind:** `21004` (CLINK Enroll)
- **Tags (request):**
  - `["p", "<service_pubkey>"]`
  - `["clink_version", "1"]`
  - `["nonce", "<counter>", "<target_difficulty>"]` — [NIP-13](https://github.com/nostr-protocol/nips/blob/master/13.md) proof of work when the service requires it (see **Proof of work**)
- **Tags (response):**
  - `["p", "<requestor_pubkey>"]`
  - `["e", "<request_event_id>"]`
  - `["clink_version", "1"]`
- **Content:** NIP-44 encrypted JSON (same pattern as Offers / Debits / Manage)

### Protocol Versioning

CLINK events utilize a mandatory `["clink_version", "1"]` tag. This ensures:
1. **Disambiguation:** Explicitly identifies events belonging to the CLINK protocol, preventing conflicts if other NIPs use the same event kind (`21004`).
2. **Version Compatibility:** Allows clients and services to verify they are using compatible versions of the CLINK protocol specification. Future versions may increment the version number (e.g., `"2"`).

Implementations MUST include this tag in both request and response events and SHOULD reject events lacking this tag or having an unsupported version number.

## Proof of work

Enroll creates an account for a new key, or returns the existing account if that key is already enrolled. Unbounded free enroll can be abused. Services **MAY** require [NIP-13](https://github.com/nostr-protocol/nips/blob/master/13.md) proof of work on the kind `21004` request to make mass scripted enrollment expensive, while a single enroll on a phone, browser tab, or low-resource agent stays interactive.

This spec does **not** mandate that every deployment enforce PoW. Operators choose based on their threat model (open public node vs invite-only vs already rate-limited). When a service *does* require PoW, the rules below apply.

### When PoW is required

1. The request event MUST include a NIP-13 `nonce` tag: `["nonce", "<counter>", "<target_difficulty>"]`.
2. The event id MUST have at least `target_difficulty` leading zero bits.
3. `target_difficulty` in the tag MUST be ≥ the service’s required difficulty (committed target — reject “lucky” high-difficulty ids that commit to a lower target).
4. If PoW is insufficient, the service SHOULD respond with error code `5` and `required_difficulty` so the client can remine once without guessing.

### Recommended difficulty

| Setting | Bits | Intent |
|---------|------|--------|
| **Recommended** | **18** | Deters scripts; typically well under ~1s on a mid-range phone / browser tab / small agent harness |
| **Stronger** | 20 | ~1M hashes; usually a few seconds on a phone |
| **Heavy** | ≥ 22 | Avoid for open public enroll aimed at low-power clients |

Expected work scales as `2^bits` SHA-256 event-id trials. One extra bit ≈ 2× wall time.

### Discover difficulty before mining

Blindly hashing at 18 and then discovering the service wants 20 wastes a full mine on weak devices.

**Beacon fast-path (recommended):** see [CLINK Beacon](clink-beacon.md). A fresh kind `30078` beacon with `enroll_difficulty` lets clients skip the probe before mining.

**Portable discovery (normative):** In the absence of a beacon, clients learn difficulty by **probing** — send an Enroll request with no `nonce` tag (or `target_difficulty` `0`). If the service requires PoW, it SHOULD respond with code `5` and `required_difficulty`; the client mines once at that value and retries. Every CLINK Enroll implementation MUST support this path. Portable clients MUST still probe when beacon is missing, stale, or not implemented.

If the probe receives no response, the client MAY retry Enroll with the recommended **18** bits as a local default. If the service does not require PoW, the probe itself is the Enroll request and a successful response completes enrollment. On code `5`, mine at `required_difficulty` and retry **once**.

**Note:** A huge difficulty from a beacon `enroll_difficulty` or a code `5` `required_difficulty` is a hang vector. Clients SHOULD NOT mine values well beyond the range above (>24): skip the beacon fast-path (probe), or return the GFY instead of remine.

Services that require PoW SHOULD return code `5` + `required_difficulty` on insufficient work so the probe path works.

Services that publish a CLINK beacon with `enroll_difficulty` MUST keep it in sync with what they enforce on kind `21004` (see [CLINK Beacon](clink-beacon.md)).

### Existing accounts

If the requestor pubkey already owns an account, a service that normally requires PoW MAY accept lower difficulty (including none) for an idempotent “return my pointers”. New account creation, when PoW is enabled, SHOULD still meet the full requirement.

## Request

Kind `21004` **Enroll request** event. Its payload is an empty object:

```json
{}
```

Services MUST associate the resulting account with the **request event’s pubkey**.

## Response (success)

```json
{
  "res": "ok",
  "noffer": "noffer1...",
  "ndebit": "ndebit1...",
  "nmanage": "nmanage1..."
}
```

| Field | Requirement |
|-------|-------------|
| `res` | `"ok"` on success |
| `noffer` | Default spontaneous (or service-default) offer for this account |
| `ndebit` | Static debit pointer for this account |
| `nmanage` | Manage pointer for this account |

Each returned bech32 string MUST follow its respective Offers, Debits, or Manage specification. In particular, returned strings MUST use:

- TLV `0` = **this service’s** pubkey
- TLV `1` = a relay the service listens on (typically the relay used for the request)
- TLV `2` = the resource-specific identifier defined by that format, where present

Idempotency: repeating Enroll with the same key MUST return equivalent pointers (bech32 strings MAY differ only if relay preference changes).

## Response (error)

Error responses use the GFY envelope (`"res": "GFY"`), consistent with other CLINK interactive response protocols.

```json
{
  "res": "GFY",
  "code": 5,
  "error": "Insufficient proof of work",
  "required_difficulty": 18
}
```

`required_difficulty` MUST be present when `code` is `5`.

Error codes (aligned with other CLINK specs):

| Code | Meaning | Expected Extra Fields |
|------|---------|-----------------------|
| 1 | Denied / not allowed | |
| 2 | Temporary Failure / Service unavailable | |
| 3 | Expired Request | `delta`: `{"max_delta_ms": 30000, "actual_delta_ms": <calculated_delta>}` |
| 4 | Rate limited | `retry_after`: `<unix_timestamp>` (optional) |
| 5 | Insufficient NIP-13 proof of work | `required_difficulty`: `<integer>` |
| 6 | Invalid Request | |

## Owner policy (normative for reference servers)

When the signer of a kind **21002** (Debit) or **21003** (Manage) request is the Nostr key that **owns** the account behind the resource pointer in that request (the same key that Enrolled):

- The service MUST allow the operation without a prior third-party debit/manage authorization grant.
- This does **not** authorize any other pubkey.

Marketplace / agent keys that are **not** the account owner still require normal Debit / Manage authorization flows.

## Client flow

```
nprofile (service) + user key
        │
        ▼
  learn difficulty:
   recommended: kind 30078 beacon → enroll_difficulty (see CLINK Beacon)
   portable: Enroll with 0 PoW → code 5 + required_difficulty
        │
        ▼
 mine if required, then kind 21004 Enroll
        │
        ▼
 noffer / ndebit / nmanage
        │
        ├── 21001 pay-into offer (others → you)
        ├── 21002 pay-out via ndebit (you → invoice)   [owner auto-allow]
        └── 21003 create offers via nmanage            [owner auto-allow]
```

## Security considerations

- Enroll proves control of a key and creates an account on the service (or returns pointers for an account that key already has). Services SHOULD rate-limit new enrolls and MAY require PoW and/or invites.
- When PoW is used, ~**18 bits** is a sensible default tradeoff. Clients SHOULD probe (code `5`) before mining so they do not double-hash on a phone.
- Returned `ndebit` is powerful for the **owner key** under owner policy; clients MUST treat the secret key as a full account credential.
- Do not overload Enroll with grant minting — that recreates ambient authority and breaks Manage’s delegation model.

## Reference Implementations

- Spec repo: this document
- Reference server: [Lightning.Pub](https://github.com/shocknet/Lightning.Pub)
- **SDK:** [CLINK SDK](https://github.com/shocknet/ClinkSDK) ([`@shocknet/clink-sdk`](https://www.npmjs.com/package/@shocknet/clink-sdk) on npm)
