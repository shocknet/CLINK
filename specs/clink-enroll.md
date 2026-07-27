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
- **Tags (response):**
  - `["p", "<requestor_pubkey>"]`
  - `["e", "<request_event_id>"]`
  - `["clink_version", "1"]`
- **Content:** NIP-44 encrypted JSON (same pattern as Offers / Debits / Manage)

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
  "code": 1,
  "error": "human readable reason"
}
```

Suggested codes (align with other CLINK specs where practical):

| Code | Meaning |
|------|---------|
| 1 | Denied / not allowed |
| 2 | Rate limited |
| 3 | Service unavailable |

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
   kind 21004 Enroll
        │
        ▼
 noffer / ndebit / nmanage
        │
        ├── 21001 pay-into offer (others → you)
        ├── 21002 pay-out via ndebit (you → invoice)   [owner auto-allow]
        └── 21003 create offers via nmanage            [owner auto-allow]
```

## Security considerations

- Enroll proves control of a key and creates (or resumes) a custodial-or-self-hosted account on the service; services SHOULD rate-limit and MAY require invites.
- Returned `ndebit` is powerful for the **owner key** under owner policy; clients MUST treat the secret key as a full account credential.
- Do not overload Enroll with grant minting — that recreates ambient authority and breaks Manage’s delegation model.

## Reference implementation

- Spec repo: this document
- Go CLI/SDK: [clink-go](https://github.com/shocknet/clink-go) (`clinkctl enroll`)
- Reference server: [Lightning.Pub](https://github.com/shocknet/Lightning.Pub)
