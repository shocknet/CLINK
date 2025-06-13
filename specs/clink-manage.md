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
  { "res": "ok", "offer_id": "<offer_id>", "details": { ... } }
  ```
- **Error**
  ```json
  { "res": "error", "code": 1, "error": "Reason" }
  ```

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