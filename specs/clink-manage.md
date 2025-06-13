# CLINK Manage Specification

## Overview

CLINK Manage defines a protocol for delegated management of wallet resources by external applications. Using a bech32-encoded pointer (`nmanage1...`), users can grant external applications specific, auditable rights to manage resources (such as offers) on their wallet server (e.g., Lightning.Pub). The protocol is extensible: this document defines the general framework and the first managed resource (offers), and provides a template for future resource types.

## Motivation

Many Lightning and Nostr use-cases require apps to manage resources on behalf of a user, but only within tightly defined boundaries. CLINK Manage provides a secure, extensible, and Nostr-native way to grant and audit such permissions, starting with offer management for marketplaces, SaaS, and other dynamic pricing scenarios.

## Pointer Format

A CLINK Manage pointer is a bech32-encoded string (per NIP-19) prefixed with `nmanage1`. The encoded TLVs are:
- `0`: 32-byte pubkey of the user's wallet server (hex encoded)
- `1`: recommended relay URL
- `2`: (optional) pointer ID (for multi-account, etc.)

## Nostr Events

- **Kind:** 21003 (CLINK Management Delegation)
- **Tags:**
  - `["p", "<wallet_server_pubkey>"]`
  - `["clink_version", "1"]`
  - `["e", "<request_event_id>"]` (in responses)
- **Content:** NIP-44 encrypted JSON payload

## Managed Resources

This section lists all currently defined managed resources. Each resource defines its own payload structure and semantics.

### 1. Offers (`offer`)

Allows an app to create, update, and delete offers on the user's wallet server.

#### Request Payloads
- **Create Offer**
  ```json
  {
    "resource": "offer",
    "action": "create",
    "offer": {
      "id": "<offer_id>",
      "price_sats": 12345,
      "description": "Product X",
      "callback_url": "https://marketplace.com/callback/123"
    }
  }
  ```
- **Update Offer**
  ```json
  {
    "resource": "offer",
    "action": "update",
    "offer": {
      "id": "<offer_id>",
      "fields": {
        "price_sats": 23456,
        "description": "Updated Product X"
      }
    }
  }
  ```
- **Delete Offer**
  ```json
  {
    "resource": "offer",
    "action": "delete",
    "offer": {
      "id": "<offer_id>"
    }
  }
  ```

#### Response Payloads
- **Success**
  ```json
  { "res": "ok", "resource": "offer", "id": "<offer_id>", "details": { ... } }
  ```
- **Error (GFY)**
  ```json
  { "res": "GFY", "code": 1, "error": "Reason" }
  ```

#### GFY (General Failure to Yield) Handling

When a request cannot be fulfilled, the wallet service MAY respond with a GFY error code.

- `1`: Request Denied (User or rule denied the request; may precede reporting)
- `2`: Temporary Failure (Wallet service issue, e.g., node offline)
- `3`: Expired Request (Request timestamp too old, e.g., >30s delta)
- `4`: Rate Limited (Requestor sending too many requests)
- `5`: Invalid Field/Value (e.g., out of range)
- `6`: Invalid Request (Malformed payload, missing fields, etc.)

A typical GFY response payload is structured as follows:

```json
{
  "res": "GFY",
  "code": <gfy_code>,
  "error": "<human_readable_error_message>"
  // Additional fields based on code (see below)
}
```

**Expected Payloads for Specific GFY Codes:**

1. **Code 1 (Request Denied):**
    ```json
    {"res": "GFY", "code": 1, "error": "Request Denied"}
    ```
2. **Code 2 (Temporary Failure):**
    ```json
    {"res": "GFY", "code": 2, "error": "Temporary Failure: <reason>"}
    ```
3. **Code 3 (Expired Request):**
    ```json
    {
      "res": "GFY", "code": 3, "error": "Expired Request",
      "delta": {"max_delta_ms": 30000, "actual_delta_ms": <calculated_delta>}
    }
    ```
4. **Code 4 (Rate Limited):**
    ```json
    {
      "res": "GFY", "code": 4, "error": "Rate Limited",
      "retry_after": <unix_timestamp> // Optional: When the client can retry
    }
    ```
5. **Code 5 (Invalid Field/Value):**
    ```json
    {
      "res": "GFY", "code": 5, "error": "Invalid Field/Value",
      "field": "price_sats", "range": { "min": 1000, "max": 1000000 } // Optional: Allowed range
    }
    ```
6. **Code 6 (Invalid Request):**
    ```json
    {"res": "GFY", "code": 6, "error": "Invalid Request: <reason>"}
    ```

#### Authorization & Ownership
- The wallet server MUST track which app created each offer and MUST reject modification or deletion requests from other apps unless explicitly permitted by the user.
- Offer IDs MUST be unique per wallet server. The wallet server is responsible for enforcing uniqueness.

#### Updatable Fields
- Only fields defined as updatable in the [CLINK Offers](clink-offers.md) spec may be included in the `fields` object for updates. Optional fields are supported as per the Offers spec.

#### Authorization Flow
- User shares their `nmanage1...` pointer with the app.
- App sends a Kind 21003 request to the wallet server.
- Wallet server prompts user for approval (or applies rules).
- On approval, the server creates/updates/deletes the offer and responds.

#### Security & Rules
- All requests are signed and auditable.
- Wallet server can enforce rules (e.g., only allow certain apps, require user approval, limit offer creation rate, etc.).
- Apps should not be able to modify or delete offers they did not create, unless explicitly permitted.

## Extensibility

New managed resources can be proposed and added using the same pointer and event kind, with a new `resource` value in the payload. Each resource should define its own payload structure, actions, and security considerations.

## Versioning

All events MUST include the `["clink_version", "1"]` tag for compatibility and future upgrades. 