# CLINK Debits Specification

## Overview

This specification defines **CLINK Debits**, a method for applications and users to request Lightning payments via another user's or application operator's **node service** using Nostr event kind `21002`. It provides static or session endpoints (`ndebit1...`) for a more fluid and secure authorization UX, complementing [CLINK Offers](clink-offers.md) by enabling the inverse operation.

## Motivation

Current approaches to payment requests either require complex and often insecure pre-provisioning steps, or introduce friction in establishing recurring payment terms. CLINK Debits leverages Nostr's native strengths to enable direct, event-driven payment requests between parties, creating opportunities for more sophisticated authorization flows and reputation-based rules.

The ideal flow is simple: A user shares their static debit pointer (e.g., via their NIP-05 address) with a service -> The service sends a request -> The user approves the request (or has pre-approved via rules) in their wallet -> A connection with clear terms is established.

## Specification

### Debit Request Pointers

A debit request pointer is a bech32 encoded string (per [NIP-19](https://github.com/nostr-protocol/nips/blob/master/19.md)) prefixed with `ndebit`. The encoded string includes the following TLV (Type-Length-Value) items:

- `0`: The 32-byte public key of the **node service** (hex encoded) — the service that receives kind `21002` events and processes debits.
- `1`: Relay URL where the node service listens for requests.
- `2`: (Optional) An opaque pointer identifier string, used by the node service to route or identify the request target (e.g., a specific budget, account, or application).
- `3`: (Optional) A session identifier (`k1`). Exactly 32 bytes of opaque binary data, used to correlate a single debit attempt (e.g., an ATM withdrawal session). MUST be generated with a cryptographically secure random number generator when present.

In **authorization flows**, the node service is the backend serving the debited user's wallet (an application sends kind `21002` debit request to them). In **session flows**, the node service is an application operator's backend (e.g., Lightning.Pub) that the user withdraws from.

If the pointer ID (TLV `2`) is omitted in the `ndebit` string, the requestor MAY assume the node service's public key itself acts as an identifier and MAY omit the `pointer` field in the request payload.

**Static vs Session ndebits:**

- A **static pointer** contains TLV items `0`–`2` only. It MAY be published in NIP-05, kind `0` metadata, or other long-lived contexts.
- A **session ndebit** is a static pointer plus a **session identifier** in TLV `3` (`k1`). It is minted per interaction (e.g., one QR per withdrawal) and MUST NOT be published as a user's primary `clink_debit`.

**Roles in session flows:** LNURL-withdraw-like use cases involve three parties, which should not be conflated:

- **Application:** the operator of the dispenser (e.g., an ATM). It mints session ndebit QRs (each embedding a fresh session identifier in TLV `3`) and displays the QR.
- **Requestor wallet:** the end user's wallet client. It scans the QR, constructs the BOLT11 invoice, and sends the kind `21002` request.
- **Node service:** the backend identified by TLV `0`. Receives kind `21002` events and processes debits per its policy.

**LNURL-withdraw-like flows:** Session ndebits address the same class of use cases as [LNURL-withdraw](https://github.com/lnurl/luds/blob/luds/03.md) (LUD-03): an application displays a QR, a requestor wallet scans it, and the node service pays out once the interaction is correlated to a specific session. In LNURL-withdraw, an ephemeral `k1` ties the wallet's invoice submission back to that session. CLINK Debits achieves the same correlation over Nostr: the application mints a session ndebit with a fresh session identifier (TLV `3`), the requestor wallet sends a kind `21002` request carrying the matching `k1`, and the node service pays the requestor's invoice. Coordinating with the application (e.g., approval before payout, notifying it of settlement) is outside the scope of this specification and depends on the node service implementation. Static pointers remain the right model for long-lived authorization (subscriptions, pre-approved debits); session ndebits are for one-shot payouts.

**Example Static Pointer Structure:**

```
ndebit1...
  0: <node_service_pubkey_hex>
  1: <relay_url>
  2: <pointer_id> (optional)
```

**Example Session ndebit Structure:**

```
ndebit1...
  0: <node_service_pubkey_hex>
  1: <relay_url>
  2: <pointer_id> (optional)
  3: <32-byte session identifier (k1)>
```

### Display and QR Encoding

The debit pointer is the complete bech32 string: `ndebit1<data>`. There is no separate URI scheme, label, or wrapper — the scanned or shared value is exactly that one string.

**QR codes:** The QR payload MUST be that string encoded as plain text (e.g. `ndebit1qvq...`). Implementations MUST NOT prepend `ndebit:`, append query parameters, use BIP-21 URIs, LNURL strings, HTTP(S) URLs, or encode TLV fields without the bech32 wrapper.

**What QR codes are not:** A debit QR is not a BOLT11 invoice. Invoices are supplied in the kind `21002` request payload and MUST NOT be substituted for the debit pointer.

**Wallet behavior:** Requestor wallets MUST recognize a scanned or pasted string that matches the bech32 `ndebit` HRP (i.e. starts with `ndebit1`) as a CLINK Debit pointer and decode it per this specification. If TLV `3` is present, the wallet MUST include a `k1` field in the kind `21002` request payload set to the lowercase hexadecimal encoding of those 32 bytes (64 characters). If TLV `3` is absent, the wallet MUST NOT invent a `k1` value.

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

Sent to the node service identified by the key and pointer.

**Example Request:**
```json
{
  "id": "<request_event_id>",
  "pubkey": "<requestor_pubkey>",
  "created_at": 1234567890,
  "kind": 21002,
  "tags": [
    ["p", "<node_service_pubkey_hex>"],
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
        "amount_sats": 10000, // Optional: node service MAY require for rules processing
        "bolt11": "<BOLT11_invoice_string>",
        "description": "<optional_app_data>",
        "k1": "<64-char lowercase hex>" // Optional: session identifier; see Session Identifiers (k1) below
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
- The node service MAY require `amount_sats` even for direct payments to process rules without decoding the invoice, but MUST verify the invoice amount upon payment.
- For budget requests, omitting `frequency` implies a one-time budget.
- A request with no `bolt11`, no `amount_sats`, and no `frequency` is implicitly a request for unrestricted access linked to the `pointer` (or the node service pubkey if `pointer` is absent), subject to node service policy and user approval.

**Session Identifiers (`k1`):**

The `k1` field correlates a kind `21002` debit request with a specific session at the receiving service (e.g., matching a pending withdrawal at an ATM to the customer's payment).

- When debiting from a **session ndebit** (ndebit with a session identifier in TLV `3`), the requestor MUST set `k1` to the lowercase hexadecimal encoding of the 32-byte TLV `3` value.
- When debiting from a **static pointer** (no TLV `3`), the requestor MUST omit `k1`.
- The node service SHOULD treat each `k1` as single-use within the scope of the target `pointer` (or node service pubkey if `pointer` is absent) and SHOULD reject or ignore reuse of a previously consumed `k1`.
- The `k1` field in the JSON payload is the hex representation; TLV `3` in the ndebit is the binary form of the same session identifier.

### Response Event

Sent by the node service upon completing or rejecting a debit request. The kind `21002` response MUST be addressed to the pubkey that signed the original request (`["p", "<requestor_pubkey>"]`).

1.  **ACK Payment Success:**
    Upon successful payment of a direct debit request, the node service sends a success response. The event itself, being signed by the node service and referencing the original request via an `e` tag, serves as a verifiable acknowledgment. The payload distinguishes between a standard Lightning payment and an internal settlement.

    - For a **standard Lightning payment**, the NIP-44 encrypted `content` MUST be: `{"res":"ok","preimage":"<lightning_preimage>"}`.
    - For an **internal settlement**, the NIP-44 encrypted `content` MUST be: `{"res":"ok"}`. The absence of a preimage indicates an internal transaction.

    The overall event structure is the same for both cases, only the encrypted `content` differs:
    ```json
    {
      "id": "<response_event_id>",
      "pubkey": "<node_service_pubkey>",
      "created_at": 1234567891,
      "kind": 21002,
      "tags": [
        ["p", "<requestor_pubkey>"],
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

When a request cannot be fulfilled, the node service MAY respond with a GFY error code.

**GFY Codes:**

- `1`: Request Denied (User or rule denied the request; may precede reporting)
- `2`: Temporary Failure (Node service issue, e.g., node offline)
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

Requestors MUST handle GFY responses gracefully.

### Protocol Versioning

CLINK events utilize a mandatory `["clink_version", "1"]` tag. This ensures:
1.  **Disambiguation:** Explicitly identifies events belonging to the CLINK protocol, preventing conflicts if other NIPs use the same event kind (`21002`).
2.  **Version Compatibility:** Allows clients and services to verify they are using compatible versions of the CLINK protocol specification. Future versions may increment the version number (e.g., `"2"`).

Implementations MUST include this tag in both request and response events and SHOULD reject events lacking this tag or having an unsupported version number.

## Process Flow Summary

### Authorization flow

1.  **Discovery**: Application obtains the user's `ndebit` pointer (e.g., via NIP-05 lookup or direct sharing).
2.  **Parsing**: Application extracts the node service pubkey, relay hint, and optional pointer ID.
3.  **Request**: Application sends a kind `21002` event to the relay, addressed to the node service pubkey.
4.  **Node Service Processing**: Node service receives the event.
    *   It authenticates the request (e.g., checks if the app pubkey is known/allowed).
    *   It evaluates the request against user rules or prompts the user for approval.
5.  **Response**: Node service sends a kind `21002` response event to the requestor pubkey.
    *   **Success (Direct Payment)**: Includes `{"res":"ok", "preimage":"..."}`.
    *   **Success (Budget Approval)**: Includes `{"res":"ok"}`.
    *   **Failure**: Includes `{"res":"GFY", ...}`.
6.  **Application Handling**: Application receives and processes the response.

### Session flow

1.  **Discovery**: Requestor wallet scans a session ndebit QR displayed by the application.
2.  **Parsing**: Requestor wallet extracts the node service pubkey, relay hint, pointer ID, and session identifier (TLV `3` / `k1`).
3.  **Request**: Requestor wallet sends a kind `21002` direct payment request (BOLT11 invoice and `k1`) to the relay, addressed to the node service pubkey.
4.  **Node Service Processing**: Node service receives the event.
    *   It processes the request per its policy (which may include out-of-band coordination with the application before paying).
5.  **Response**: Node service sends a kind `21002` response event to the requestor wallet pubkey.
    *   **Success (Direct Payment)**: Includes `{"res":"ok", "preimage":"..."}`.
    *   **Failure**: Includes `{"res":"GFY", ...}`.

## Implementation Guidance

### Node Service

**MUST:**
- Listen for kind `21002` events on specified relays (or relays user configures).
- Validate incoming requests.
- Send kind `21002` responses (`ok` or `GFY`).
- Process Lightning payments securely for approved direct payment requests.

**SHOULD:**
- Provide a UI for users to manage permissions, budgets, and rules.
- Distinguish between direct payment and budget requests in approval prompts.
- Track pending and consumed `k1` values per `pointer` and reject duplicate session identifiers.
- Handle request idempotency or replacement (e.g., only process the latest request from a given app pubkey if multiple are pending).
- Implement robust budget tracking (amount, frequency resets).
- Consider adding fee reserves to budgets based on policy.
- Implement automatic approval/denial based on user-defined rules (e.g., allow app X up to Y sats per month).

### Wallet Client (UI)

**MUST:**
- Display notifications/prompts for pending requests requiring user approval (authorization flows).
- When initiating a debit from a decoded session ndebit, include `k1` in the kind `21002` payload as specified in Session Identifiers (`k1`).
- Listen for kind `21002` responses to session ndebit requests and surface failures to the user.

**SHOULD:**
- Show completed CLINK Debit payments in transaction history.
- Provide UI for managing rules, permissions, and budgets associated with CLINK Debits.
- Display context about the requesting application's pubkey (NIP-05, WoT status) to aid user decisions.

**MAY:**
- Distinguish request types (direct payment vs. budget) in UI elements.
- Implement reporting features (e.g., NIP-56) to flag potentially abusive applications.

### Application

**MUST (authorization flows):**
- Obtain the user's `ndebit` pointer.
- Send well-formed kind `21002` request events.
- Listen for kind `21002` response events via Nostr subscriptions.
- Handle `ok` and `GFY` responses appropriately.

**MUST (session flows):**
- When minting session ndebit QRs, generate a fresh 32-byte session identifier (TLV `3`) per session and encode a new `ndebit1...` string for each QR.

**SHOULD:**
- Make the requesting pubkey, event ID, or signature easily verifiable by the user/wallet client.
- Handle scenarios where a previous request might be superseded before a response is received.
- Consider using budget requests even for one-time payments if invoice expiry or retries are concerns (allows node service more flexibility).
- Treat each `k1` as single-use at the application layer; do not issue a new session QR for a retry while a prior session is still pending settlement.

## Security Considerations

1.  **Encryption**: All `content` payloads MUST use NIP-44 encryption between the requestor and node service.
2.  **Authorization**: Node services MUST implement strong authorization logic. Permissions should be granular (per-app pubkey, per-pointer ID if used) and constrained by user-approved budgets/rules.
3.  **User Education**: Users must understand the implications of granting permissions, especially for automatic approvals or recurring budgets.
4.  **Abuse Prevention**: Node services should consider rate limiting, reputation tracking (e.g., NIP-56 integration), or other mechanisms to discourage spam/abuse.
5.  **Atomic Operations**: Node services should ensure payment processing and budget deduction are atomic to prevent race conditions or overspending.
6.  **Session Identifiers**: Node services that accept `k1` SHOULD treat each value as single-use within the scope of the target `pointer`. Implementations that correlate sessions with an application (e.g., an ATM) MUST NOT rely on amount alone; approval coordination is outside the scope of this specification and typically done with a management RPC.

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
- Node services MUST treat updated budget requests that exceed prior auto-approval limits as new requests requiring user confirmation.

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

- **Node service:** [Lightning.Pub](https://github.com/shocknet/Lightning.Pub)
- **Wallet Client:** [ShockWallet](https://shockwallet.app)
- **SDK:** [CLINK SDK](https://github.com/shocknet/ClinkSDK)
- **Demo Client:** [clinkme.dev](https://clinkme.dev/)
