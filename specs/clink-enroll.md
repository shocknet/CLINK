# CLINK Enroll Specification

## Overview

**CLINK Enroll** binds a Nostr key to an account (pointer) on a node service and returns that account’s default static CLINK pointers (`noffer1…`, `ndebit1…`, `nmanage1…`).

It is the bootstrap step for headless clients, CLIs, and agents that act **as** the user (direct-use principal). It does **not** grant third parties spend or manage rights.

## Motivation

Offers, Debits, and Manage all require a service pubkey, relay, and account pointer. Without Enroll, clients must use proprietary wallet RPC (e.g. Lightning.Pub `GetUserInfo`) to create an account and learn those pointers.

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

There is no separate “app” pubkey in the pointer model. Account identity on the service is an opaque **pointer** (user id) chosen by the service.

## Nostr Events

- **Kind:** `21004` (CLINK Enroll)
- **Tags (request):**
  - `["p", "<service_pubkey>"]`
  - `["clink_version", "1"]`
  - `["nonce", "<counter>", "<target_difficulty>"]` — [NIP-13](https://github.com/nostr-protocol/nips/blob/master/13.md) proof of work (see **Proof of work**)
- **Tags (response):**
  - `["p", "<requestor_pubkey>"]`
  - `["e", "<request_event_id>"]`
  - `["clink_version", "1"]`
- **Content:** NIP-44 encrypted JSON (same pattern as Offers / Debits / Manage)

## Proof of work

Enroll creates (or resumes) an account. Unbounded free enroll is a DoS / spam vector. Request events MUST carry [NIP-13](https://github.com/nostr-protocol/nips/blob/master/13.md) proof of work so mass scripted enrollment is expensive, while a single enroll on a phone, browser tab, or low-resource agent stays interactive.

### Rules

1. The kind `21004` **request** event MUST include a NIP-13 `nonce` tag: `["nonce", "<counter>", "<target_difficulty>"]`.
2. The event id MUST have at least `target_difficulty` leading zero bits.
3. `target_difficulty` in the tag MUST be ≥ the service’s required difficulty (committed target — services MUST reject “lucky” high-difficulty ids that commit to a lower target).
4. Services MUST reject requests that fail (1)–(3), preferably with error code `4` and `required_difficulty` so clients can remine once.

### Recommended difficulty

| Setting | Bits | Intent |
|---------|------|--------|
| **Minimum allowed** | 16 | Floor for public enroll (~65k hashes expected) |
| **Recommended default** | **18** | Deterrent for naive scripts; typically well under ~1s on a mid-range phone / browser WASM / small VPS agent |
| **Stronger public** | 20 | ~1M hashes expected; still usually a few seconds on a phone, not “forever” |
| **Avoid by default** | ≥ 22 | Fine for invite-gated or desktop-only services; too slow for shitty phones / casual web tabs |

Expected work scales as `2^bits` SHA-256 event-id trials. One extra bit ≈ 2× wall time.

Reference servers SHOULD default to **18** and MAY raise toward **20** under load. They MUST NOT require ≥ 22 for open public enroll unless they document that low-power clients are unsupported or offer an invite / lower-difficulty path.

### Existing accounts

If the requestor pubkey already owns an account on the service, the service MAY:

- accept the same PoW requirement, or
- accept a lower difficulty (including `0`) for idempotent “return my pointers” re-enroll

New account creation MUST still meet the full required difficulty.

### Advertising difficulty

Services SHOULD advertise `required_difficulty` so clients mine once:

- in error responses when PoW is insufficient (see below), and/or
- out of band (e.g. service kind `0` / documentation)

Clients SHOULD mine at the advertised value (or the recommended default **18** if unknown), and on code `4` remine at `required_difficulty`.

## Request

Empty object or optional fields:

```json
{}
```

Optional:

```json
{
  "preferred_pointer": "<hint>"
}
```

Services MAY ignore `preferred_pointer`. Services MUST associate the resulting account with the **request event’s pubkey**.

## Response (success)

```json
{
  "pointer": "<account_pointer>",
  "noffer": "noffer1...",
  "ndebit": "ndebit1...",
  "nmanage": "nmanage1..."
}
```

| Field | Requirement |
|-------|-------------|
| `pointer` | Opaque account id used as TLV `2` in the returned bech32s |
| `noffer` | Default spontaneous (or service-default) offer for this account |
| `ndebit` | Static debit pointer for this account |
| `nmanage` | Manage pointer for this account |

Returned bech32s MUST use:

- TLV `0` = **this service’s** pubkey
- TLV `1` = a relay the service listens on (typically the relay used for the request)
- TLV `2` = `pointer` (where the format includes a pointer)

Idempotency: repeating Enroll with the same key MUST return the same `pointer` and equivalent pointers (bech32 strings MAY differ only if relay preference changes).

## Response (error)

```json
{
  "code": 4,
  "error": "insufficient proof of work",
  "required_difficulty": 18
}
```

`required_difficulty` MUST be present when `code` is `4`.

Suggested codes (align with other CLINK specs where practical):

| Code | Meaning |
|------|---------|
| 1 | Denied / not allowed |
| 2 | Rate limited |
| 3 | Service unavailable |
| 4 | Insufficient NIP-13 proof of work (see `required_difficulty`) |

## Owner policy (normative for reference servers)

When the signer of a kind **21002** (Debit) or **21003** (Manage) request is the Nostr key that **owns** the account identified by the pointer (the same key that Enrolled):

- The service MUST allow the operation without a prior third-party debit/manage authorization grant.
- This does **not** authorize any other pubkey.

Marketplace / agent keys that are **not** the account owner still require normal Debit / Manage authorization flows.

## Client flow

```
nprofile (service) + user key
        │
        ▼
 mine NIP-13 PoW (default 18 bits) on kind 21004
        │
        ▼
   kind 21004 Enroll
        │
        ├─ code 4 → remine at required_difficulty → retry
        ▼
 noffer / ndebit / nmanage
        │
        ├── 21001 pay-into offer (others → you)
        ├── 21002 pay-out via ndebit (you → invoice)   [owner auto-allow]
        └── 21003 create offers via nmanage            [owner auto-allow]
```

## Security considerations

- Enroll proves control of a key and creates (or resumes) a custodial-or-self-hosted account on the service; services SHOULD rate-limit **and** require NIP-13 PoW for new accounts. PoW alone is not enough under a well-funded attacker — combine with rate limits / invites.
- Recommended **18-bit** PoW aims for “annoying to spray thousands of accounts, fine for one human or agent on a phone.” Do not push public defaults into “wait forever on a weak device” territory (≥ 22).
- Returned `ndebit` is powerful for the **owner key** under owner policy; clients MUST treat the secret key as a full account credential.
- Do not overload Enroll with grant minting — that recreates ambient authority and breaks Manage’s delegation model.

## Reference implementation

- Spec repo: this document
- Go CLI/SDK: [clink-go](https://github.com/shocknet/clink-go) (`clinkctl enroll`)
- Reference server: [Lightning.Pub](https://github.com/shocknet/Lightning.Pub)
