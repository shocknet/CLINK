# CLINK Debits Specification

## Overview

This specification defines **CLINK Debits**, a method for applications and services to request Lightning payments from a user's wallet service using Nostr event kind `21002`. It provides users with a static endpoint (`ndebit1...`) for a more fluid and secure authorization UX, complementing [CLINK Offers](clink-offers.md) by enabling the inverse operation.

## Motivation

Current approaches to payment requests either require complex and often insecure pre-provisioning steps, or introduce friction in establishing recurring payment terms. CLINK Debits leverages Nostr's native strengths to enable direct, event-driven payment requests between parties, creating opportunities for more sophisticated authorization flows and reputation-based rules.

The ideal flow is simple: A user shares their static debit pointer (e.g., via their NIP-05 address) with a service -> The service sends a request -> The user approves the request (or has pre-approved via rules) in their wallet -> A connection with clear terms is established.

## Specification

### Debit Request Pointers

A debit request pointer is a bech32 encoded string (per [NIP-19](https://github.com/nostr-protocol/nips/blob/master/19.md)) prefixed with `ndebit`. The encoded string includes the following TLV (Type-Length-Value) items:

- `0`: The 32-byte public key of the user's wallet service (hex encoded).
- `1`: A recommended relay URL where the wallet service listens for requests.
- `2`: (Optional) An opaque pointer identifier string, used by the wallet service to route or identify the request target (e.g., a specific budget or account).

If the pointer ID (TLV `2`) is omitted in the `ndebit` string, the requestor MAY assume the wallet service's public key itself acts as the identifier and MAY omit the `pointer` field in the request payload.

**Example Pointer Structure:**

```
ndebit1...
  0: <wallet_service_pubkey_hex>
  1: <relay_url>
  2: <pointer_id> (optional)
```

## Integration with Nostr

### NIP-01 User Metadata

Users can advertise their primary debit pointer in their kind `0` metadata event using a `clink_debit` field (or similar agreed-upon field name) to allow applications default awareness of their payment source.

**Example:**
```json
{
  "pubkey": "hex_pub",
  "kind": 0,
  "content": "{\"name\": \"bob\", \"nip05\": \"bob@example.com\", \"clink_debit\": \"ndebit1...\"}"
  // ...
}
```
*(Note: The exact field name, like `clink_debit` or `nip68` if backward compatibility is desired, should be finalized)*

### NIP-05 Lookup

To simplify connecting apps to wallets via NIP-05 identifiers (like Lightning Addresses), NIP-05 servers can include a mapping for debit pointers using the `clink_debit` field.

**Example NIP-05 Response:**
```json
{
  "names": {
    "bob": "<hex_pub>"
  },
  "clink_debit": {
    "bob": "ndebit1..."
  }
}
```

## Nostr Events (Kind 21002)

CLINK Debits uses ephemeral event kind `21002` for requests and responses.

**Common Structure:**
- `kind`: `21002`
- `content`: NIP-44 encrypted payload.
- `tags`:
  - `["p", "<recipient_pubkey_hex>"]`: The pubkey receiving the event.
  - `["e", "<request_event_id>"]`: (In responses only) The event ID of the request being responded to.

### Debit Request Event

Sent by the application/service to the user's wallet service.

**Example Request:**
```json
{
  "id": "<request_event_id>",
  "pubkey": "<application_pubkey>",
  "created_at": 1234567890,
  "kind": 21002,
  "tags": [
    ["p", "<wallet_service_pubkey_hex>"],
    ["clink_version", "1"]
  ],
  "content": "<NIP-44 encrypted request payload>",
  "sig": "<signature>"
}
```

**Decrypted Request Payload (`content`)**: Must contain one of the following JSON structures:

1.  **Direct Payment Request:**
    ```json
    {
        "pointer": "<pointer_id>", // Optional, from ndebit TLV 2
        "amount_sats": 10000, // Optional: Wallet MAY require for rules processing
        "bolt11": "<BOLT11_invoice_string>",
        "description": "<optional_app_data>"
    }
    ```

2.  **Budget Request:**
    ```json
    {
        "pointer": "<pointer_id>", // Optional, from ndebit TLV 2
        "amount_sats": 50000,
        "frequency": {
            "number": 1,
            "unit": "month" // Possible values: "day", "week", "month"
        },
        "description": "<optional_app_data>"
    }
    ```

**Notes on Request Payload:**
- The wallet service MAY require `amount_sats` even for direct payments to process rules without decoding the invoice, but MUST verify the invoice amount upon payment.
- For budget requests, omitting `frequency` implies a one-time budget.
- A request with no `bolt11`, no `amount_sats`, and no `frequency` is implicitly a request for unrestricted access linked to the `pointer` (or the wallet pubkey if `pointer` is absent), subject to wallet service policy and user approval.

### Response Event

Sent by the wallet service back to the application/service.

1.  **ACK Payment Success:**
    Upon successful payment of a direct debit request, the wallet service sends a success response. The event itself, being signed by the wallet service and referencing the original request via an `e` tag, serves as a verifiable acknowledgment. The payload distinguishes between a standard Lightning payment and an internal settlement.

    - For a **standard Lightning payment**, the NIP-44 encrypted `content` MUST be: `{"res":"ok","preimage":"<lightning_preimage>"}`.
    - For an **internal settlement**, the NIP-44 encrypted `content` MUST be: `{"res":"ok"}`. The absence of a preimage indicates an internal transaction.

    The overall event structure is the same for both cases, only the encrypted `content` differs:
    ```json
    {
      "id": "<response_event_id>",
      "pubkey": "<wallet_service_pubkey>",
      "created_at": 1234567891,
      "kind": 21002,
      "tags": [
        ["p", "<application_pubkey>"],
        ["e", "<request_event_id>"],
        ["clink_version", "1"]
      ],
      "content": "<NIP-44 encrypted payload>",
      "sig": "<signature>"
    }
    ```

2.  **ACK (Budget Approval Success):**
    ```json
    {
      // ... similar structure ...
      "content": "<NIP-44 encrypted {\"res\":\"ok\"}>",
      "sig": "<signature>"
    }
    ```

3.  **GFY (General Failure to Yield) Response:**
    ```json
    {
      // ... similar structure ...
      "content": "<NIP-44 encrypted {\"res\":\"GFY\",\"code\":<code>,\"error\":\"<message>\", ...}>",
      "sig": "<signature>"
    }
    ```

## GFY (General Failure to Yield) Handling

When a request cannot be fulfilled, the wallet service MAY respond with a GFY error code.

**GFY Codes:**

- `1`: Request Denied (User or rule denied the request; may precede reporting)
- `2`: Temporary Failure (Wallet service issue, e.g., node offline)
- `3`: Expired Request (Request timestamp too old, e.g., >30s delta)
- `4`: Rate Limited (Requestor sending too many requests)
- `5`: Invalid Amount (Amount outside acceptable range or budget)
- `6`: Invalid Request (Malformed payload, missing fields, etc.)

**GFY Response Payload Structure (Decrypted `content`):**
```json
{
  "res": "GFY",
  "code": <gfy_code>,
  "error": "<human_readable_error_message>"
  // Additional fields based on code (see below)
}
```

**Expected Payloads for Specific GFY Codes:**

1.  **Code 1 (Request Denied):**
    ```json
    {"res": "GFY", "code": 1, "error": "Request Denied"}
    ```
2.  **Code 2 (Temporary Failure):**
    ```json
    {"res": "GFY", "code": 2, "error": "Temporary Failure: <reason>"}
    ```
3.  **Code 3 (Expired Request):**
    ```json
    {
      "res": "GFY", "code": 3, "error": "Expired Request",
      "delta": {"max_delta_ms": 30000, "actual_delta_ms": <calculated_delta>}
    }
    ```
4.  **Code 4 (Rate Limited):**
    ```json
    {
      "res": "GFY", "code": 4, "error": "Rate Limited",
      "retry_after": <unix_timestamp> // Optional: When the client can retry
    }
    ```
5.  **Code 5 (Invalid Amount):**
    ```json
    {
      "res": "GFY", "code": 5, "error": "Invalid Amount",
      "range": {"min": <min_sats>, "max": <max_sats>} // Optional: Allowed range
    }
    ```
6.  **Code 6 (Invalid Request):**
    ```json
    {"res": "GFY", "code": 6, "error": "Invalid Request: <reason>"}
    ```

Applications MUST handle GFY responses gracefully.

### Protocol Versioning

CLINK events utilize a mandatory `["clink_version", "1"]` tag. This ensures:
1.  **Disambiguation:** Explicitly identifies events belonging to the CLINK protocol, preventing conflicts if other NIPs use the same event kind (`21002`).
2.  **Version Compatibility:** Allows clients and services to verify they are using compatible versions of the CLINK protocol specification. Future versions may increment the version number (e.g., `"2"`).

Implementations MUST include this tag in both request and response events and SHOULD reject events lacking this tag or having an unsupported version number.

## Process Flow Summary

1.  **Discovery**: Application obtains the user's `ndebit` pointer (e.g., via NIP-05 lookup or direct sharing).
2.  **Parsing**: Application extracts the wallet service pubkey, relay hint, and optional pointer ID.
3.  **Request**: Application sends a kind `21002` event (direct payment or budget request) to the relay, addressed to the wallet service pubkey.
4.  **Wallet Service Processing**: Wallet service receives the event.
    *   It authenticates the request (e.g., checks if the app pubkey is known/allowed).
    *   It evaluates the request against user rules or prompts the user for approval.
5.  **Response**: Wallet service sends a kind `21002` response event back to the application pubkey.
    *   **Success (Direct Payment)**: Includes `{"res":"ok", "preimage":"..."}`.
    *   **Success (Budget Approval)**: Includes `{"res":"ok"}`.
    *   **Failure**: Includes `{"res":"GFY", ...}`.
6.  **Application Handling**: Application receives and processes the response.

## Implementation Guidance

### Wallet Service

**MUST:**
- Listen for kind `21002` events on specified relays (or relays user configures).
- Validate incoming requests.
- Send kind `21002` responses (`ok` or `GFY`).
- Process Lightning payments securely for approved direct payment requests.

**SHOULD:**
- Provide a UI for users to manage permissions, budgets, and rules.
- Distinguish between direct payment and budget requests in approval prompts.
- Handle request idempotency or replacement (e.g., only process the latest request from a given app pubkey if multiple are pending).
- Implement robust budget tracking (amount, frequency resets).
- Consider adding fee reserves to budgets based on policy.
- Implement automatic approval/denial based on user-defined rules (e.g., allow app X up to Y sats per month).

### Wallet Client (UI)

**MUST:**
- Display notifications/prompts for pending requests requiring user approval.

**SHOULD:**
- Show completed CLINK Debit payments in transaction history.
- Provide UI for managing rules, permissions, and budgets associated with CLINK Debits.
- Display context about the requesting application's pubkey (NIP-05, WoT status) to aid user decisions.

**MAY:**
- Distinguish request types (direct payment vs. budget) in UI elements.
- Implement reporting features (e.g., NIP-56) to flag potentially abusive applications.

### Application

**MUST:**
- Obtain the user's `ndebit` pointer.
- Send well-formed kind `21002` request events.
- Listen for kind `21002` response events via Nostr subscriptions.
- Handle `ok` and `GFY` responses appropriately.

**SHOULD:**
- Make the requesting pubkey, event ID, or signature easily verifiable by the user/wallet client.
- Handle scenarios where a previous request might be superseded before a response is received.
- Consider using budget requests even for one-time payments if invoice expiry or retries are concerns (allows wallet service more flexibility).

## Security Considerations

1.  **Encryption**: All `content` payloads MUST use NIP-44 encryption between the application and wallet service.
2.  **Authorization**: Wallet services MUST implement strong authorization logic. Permissions should be granular (per-app pubkey, per-pointer ID if used) and constrained by user-approved budgets/rules.
3.  **User Education**: Users must understand the implications of granting permissions, especially for automatic approvals or recurring budgets.
4.  **Abuse Prevention**: Wallet services should consider rate limiting, reputation tracking (e.g., NIP-56 integration), or other mechanisms to discourage spam/abuse.
5.  **Atomic Operations**: Wallet services should ensure payment processing and budget deduction are atomic to prevent race conditions or overspending.

## Handling Fluctuating Amounts (e.g., Fiat Pricing)

When amounts are pegged to volatile assets (like fiat currencies), careful handling is needed.

### Client-side (Wallet)
- Allow users to define rules/budgets in fiat terms.
- The wallet client SHOULD record the exchange rate at the time of rule creation.
- It MAY periodically check rates and alert the user if a rule's satoshi equivalent drifts significantly, suggesting a rule update.
- Any dynamic calculation influencing an approval MUST be clearly communicated to the user.

### Service-side (Application)
- Services MAY send updated budget requests if exchange rates change significantly.
- Updated requests MUST use the same `pointer` value to link to the existing budget.
- Wallet services MUST treat updated budget requests that exceed prior auto-approval limits as new requests requiring user confirmation.

**Example Updated Budget Request Payload:**
```json
{
    "pointer": "<original_pointer_id>",
    "amount_sats": 1050000, // New calculated amount
    "frequency": {"number": 1, "unit": "month"},
    "description": "Updated subscription: $50 = 1050000 sats (previously 1000000 sats)" // Optional context
}
```

## Reference Implementations

- **Wallet Node:** [Lightning.Pub](https://github.com/shocknet/Lightning.Pub)
- **Wallet Client:** [ShockWallet](https://shockwallet.app)
- **SDK:** [CLINK SDK](https://github.com/shocknet/ClinkSDK)
- **Demo Client:** [clinkme.dev](https://clinkme.dev/)