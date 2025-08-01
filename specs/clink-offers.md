# CLINK Offers Specification

## Overview

This specification defines **CLINK Offers**, a format for static payment codes in Nostr (`noffer1...`). It serves as a Nostr-native successor to LNURL-Pay, enabling users to initiate Lightning Network payments by scanning or clicking a string, without relying on traditional web servers.

## Motivation

Current Lightning payment flows either require maintaining HTTP endpoints, leading to unnecessary complexity and centralization risks in self-hosted scenarios, or depend on slow and unreliable P2P transport mechanisms that are impractical for web applications. By leveraging Nostr's native capabilities for messaging and encryption, CLINK Offers provides a more direct and robust alternative that eliminates these dependencies.

This approach enables truly spontaneous payments that work seamlessly across all platforms, with consistent performance and reliability regardless of the application environment.

## Specification

### Static Payment Code Format (`noffer`)

The static payment code is a bech32 encoded string (per [NIP-19](https://github.com/nostr-protocol/nips/blob/master/19.md)) prefixed with `noffer`. The encoded string includes the following TLV (Type-Length-Value) items:

- `0`: The 32-byte public key of the receiving service (hex encoded).
- `1`: A recommended relay URL where the receiving service listens for payment requests.
- `2`: An opaque offer identifier string (defined by the receiving service).
- `3`: (Optional) A flag indicating the pricing type:
  - `0`: Fixed price (Price stated in TLV `4`)
  - `1`: Variable price (Price determined by the service upon request, e.g., based on fiat conversion)
  - `2`: Spontaneous payment (Payer specifies the amount in the request)
- `4`: (Optional) The price in satoshis (integer) primarily for display purposes or fixed price offers.
- `5`: (Optional) Currency code (e.g., "USD", "EUR") if the price in TLV 4 represents a non-satoshi amount. If present, pricing type `1` (Variable) MUST be used.

**Default Behavior:** If neither price (TLV `4`) nor pricing type (TLV `3`) is present, the offer SHOULD be treated as type `2` (Spontaneous payment).

**Amount Unit:** Amounts in TLV `4` and request/response payloads are specified in **satoshis**.

**Example Structure:**
```
noffer1...
  0: <receiver_public_key_hex>
  1: <relay_url>
  2: <offer_id_string>
  3: <pricing_type_flag> (0, 1, or 2, optional)
  4: <price_in_sats> (integer, optional)
  5: <currency_code> (string, optional, requires type 1)
```

## Integration with Nostr

### NIP-01 User Metadata

Users or services can advertise a default/primary offer (typically for spontaneous payments) in their kind `0` metadata event using a `clink_offer` field (or similar).

**Example:**
```json
{
  "pubkey": "<hex_pub>",
  "kind": 0,
  "content": "{\"name\": \"bob\", \"nip05\": \"bob@example.com\", \"clink_offer\": \"noffer1...\"}"
  // ...
}
```

### NIP-05 Lookup ("Lightning Addresses")

To support trust-minimized Lightning Address functionality, NIP-05 servers can include a mapping for offers (typically spontaneous payment offers) using the `clink_offer` field.

**Example NIP-05 Response:**
```json
{
  "names": {
    "bob": "<hex_pub>"
  },
  "clink_offer": {
    "bob": "noffer1..."
  }
}
```

### NIP-57 Zaps

CLINK Offers can replace the LNURL-pay callback step in the NIP-57 zap flow:

1.  The zapping client creates a kind `9734` zap request event.
2.  Instead of hitting an LNURL-P endpoint, the client sends a CLINK Offer payment request (Kind `21001`, see below) to the recipient's service pubkey found in the `noffer`.
3.  The encrypted payload of the Kind `21001` request includes both the `offer` identifier and the full kind `9734` event as a stringified JSON object in a `zap` field.
4.  The receiving service decrypts the Kind `21001` content, extracts the kind `9734` event, validates it, and uses its details (amount, tags) to generate the invoice.
5.  The receiver responds with the invoice via a Kind `21001` response event.
6.  Once the payer pays the invoice, the receiver generates and publishes the kind `9735` zap receipt as specified in NIP-57.

**Recommendation for Zap Offers:** Services supporting zaps SHOULD use an offer identifier starting with `zap` (e.g., `zap_default`, `zap_profile`) in their `noffer` string. Clients seeing this prefix can be confident in sending the `zap` payload. Services receiving a request for a zap-prefixed offer *without* a `zap` payload SHOULD treat it as a standard spontaneous payment.

## Nostr Events (Kind 21001)

CLINK Offers uses ephemeral event kind `21001` for requests and responses.

**Common Structure:**
- `kind`: `21001`
- `content`: NIP-44 encrypted payload.
- `tags`:
  - `["p", "<recipient_pubkey_hex>"]`: The pubkey receiving the event.
  - `["e", "<request_event_id>"]`: (In responses only) The event ID of the request being responded to.

### Offer Request Event

Sent by the payer's wallet to the receiving service.

**Example Request:**
```json
{
  "id": "<request_event_id>",
  "pubkey": "<payer_pubkey>", // Can be an ephemeral key for privacy
  "created_at": 1234567890,
  "kind": 21001,
  "tags": [
    ["p", "<receiver_service_pubkey_hex>"],
    ["clink_version", "1"]
  ],
  "content": "<NIP-44 encrypted request payload>",
  "sig": "<signature>"
}
```

**Decrypted Request Payload (`content`)**: A JSON object with the following fields:

```json
{
    "offer": "<offer_id_string>", // From noffer TLV 2
    "amount_sats": <amount_in_sats_integer>, // Required for spontaneous/variable, optional otherwise
    "payer_data": { ... }, // Optional: Arbitrary JSON object with payer info (e.g., NIP-05, name, pubkey)
    "zap": "{...}" // Optional: Stringified JSON of kind 9734 zap request event for NIP-57 flow
}
```

### Offer Response Event

Sent by the receiving service back to the payer.

1.  **Success Response (Invoice):**
    ```json
    {
      "id": "<response_event_id>",
      "pubkey": "<receiver_service_pubkey>",
      "created_at": 1234567891,
      "kind": 21001,
      "tags": [
        ["p", "<payer_pubkey>"],
        ["e", "<request_event_id>"],
        ["clink_version", "1"]
      ],
      "content": "<NIP-44 encrypted {\"bolt11\":\"<BOLT11_invoice_string>\"}>",
      "sig": "<signature>"
    }
    ```

2.  **Error Response:**
    ```json
    {
      "id": "<response_event_id>",
      "pubkey": "<receiver_service_pubkey>",
      "created_at": 1234567891,
      "kind": 21001,
      "tags": [
        ["p", "<payer_pubkey>"],
        ["e", "<request_event_id>"],
        ["clink_version", "1"]
      ],
      "content": "<NIP-44 encrypted {\"error\":\"<error_message>\",\"code\":<error_code>,\"range\":{\"min\":<min_sats>,\"max\":<max_sats>}}>",
      "sig": "<signature>"
    }
    ```
    *Common reasons might include: an invalid offer (code 1), an expired or moved offer (code 3), or an invalid amount (code 5).*

## Error Handling

To ensure consistent error handling across implementations, this NIP defines the following error responses, similar to other NIPs. Clients MUST handle these error responses gracefully and display appropriate messages to users.

### Error Codes

- `1`: **Invalid Offer**: The requested offer ID is invalid or no longer available.
- `2`: **Temporary Failure**: The receiver is temporarily unable to process the request.
- `3`: **Expired or Moved Offer**: The offer has expired, been replaced, or permanently moved.
- `4`: **Unsupported Feature**: The receiver doesn't support a feature requested by the payer.
- `5`: **Invalid Amount**: The amount specified is too big or too small.

### Error Response Payloads

Error responses are sent as kind `21001` events with an encrypted `content` field containing a JSON object. The base structure is:

```json
{
  "error": "<human_readable_error_message>",
  "code": <error_code>
  // Additional fields may be included depending on the error type.
}
```

#### Code 1: Invalid Offer
- **Payload**: `{"error": "Invalid Offer", "code": 1}`

#### Code 2: Temporary Failure
- **Payload**: `{"error": "Temporary Failure", "code": 2}`

#### Code 3: Expired or Moved Offer
When an offer has expired, been renewed, or permanently moved, the service SHOULD respond with `code: 3`.

If a new version of the offer is available (e.g., renewed or moved), the service SHOULD include a `latest` field containing the new `noffer1...` string.

A client receiving a `code: 3` response should check for the presence of the `latest` field.
- If `latest` is present, the client SHOULD update its stored offer and automatically retry the request.
- If `latest` is not present, the client MUST treat the offer as permanently expired.

- **Payload (with forwarding):**
  ```json
  {
    "error": "Offer has been replaced or moved.",
    "code": 3,
    "latest": "<new_noffer1...>"
  }
  ```
- **Payload (without forwarding):**
  ```json
  {
    "error": "Offer has expired.",
    "code": 3
  }
  ```

#### Code 4: Unsupported Feature
- **Payload**: `{"error": "Unsupported Feature", "code": 4}`

#### Code 5: Invalid Amount
The response SHOULD include the acceptable `range` for the amount in sats.
- **Payload**:
```json
{
  "error": "Invalid Amount",
  "code": 5,
  "range": { "min": 10, "max": 10000000 }
}
```

### Protocol Versioning

CLINK events utilize a mandatory `["clink_version", "1"]` tag. This ensures:
1.  **Disambiguation:** Explicitly identifies events belonging to the CLINK protocol, preventing conflicts if other NIPs use the same event kind (`21001`).
2.  **Version Compatibility:** Allows clients and services to verify they are using compatible versions of the CLINK protocol specification. Future versions may increment the version number (e.g., `"2"`).

Implementations MUST include this tag in both request and response events and SHOULD reject events lacking this tag or having an unsupported version number.

## General Process Flow

1.  **Initiation**: Payer scans/clicks `noffer` string or uses NIP-05 lookup.
2.  **Decoding**: Payer's wallet decodes the `noffer` to get service pubkey, relay hint, offer ID, and pricing info.
3.  **Request**: Payer's wallet constructs and sends a Kind `21001` request event to the relay, addressed to the service pubkey.
    *   Includes `offer` ID.
    *   Includes `amount_sats` if offer type is spontaneous (`2`) or variable (`1`).
    *   Optionally includes `payer_data`.
    *   Optionally includes `zap` payload if performing a NIP-57 zap.
4.  **Service Processing**: Receiving service listens for Kind `21001` events.
    *   Decrypts payload.
    *   Validates the `offer` ID and `amount_sats` against offer parameters.
    *   (If variable price) Calculates the current price in sats.
    *   (If zap) Processes the `zap` event.
    *   Generates a BOLT11 Lightning invoice.
5.  **Response**: Service sends a Kind `21001` response event back to the payer's pubkey.
    *   Success: Encrypted payload contains `{"bolt11":"..."}`.
    *   Failure: Encrypted payload contains `{"error":"...","code":...,"range":{"min":...,"max":...}}`.
6.  **Payment**: Payer's wallet receives the response, decrypts it.
    *   If `ok`, presents the invoice to the user for payment (or pays automatically via NWC/CLINK Debits etc.).
    *   If `error`, displays the reason to the user.
7.  **Receipt (Optional)**: After successful payment, the receiving service MAY provide a receipt. If the original request was a NIP-57 zap, the service generates and publishes a `kind: 9735` zap receipt. For other interactions, it MAY send a direct `kind: 21001` Payment Receipt event (see below) to the payer.

## Implementation Guidance

### Payer Wallet
- MUST support decoding `noffer` strings.
- MUST support sending Kind `21001` requests with NIP-44 encryption.
- MUST support receiving Kind `21001` responses and decrypting them.
- SHOULD handle different offer types (fixed, variable, spontaneous).
- SHOULD support NIP-57 zap flow integration by including the `zap` payload.
- MAY use ephemeral keys for requests for enhanced privacy.

### Receiving Service
- MUST generate `noffer` strings correctly.
- MUST listen for Kind `21001` requests on its advertised relay.
- MUST validate requests against offer parameters.
- MUST generate BOLT11 invoices.
- MUST send Kind `21001` responses (success or error) with NIP-44 encryption.
- SHOULD handle zap requests according to NIP-57 if supporting zaps.

## Security & Privacy Considerations
- Use NIP-44 for all content encryption.
- Payer wallets can use ephemeral keys for requests to avoid linking payments to a primary Nostr identity.
- Receiving services should be mindful of potential rate-limiting or abuse vectors on their listening relay.
- Consider using gift-wrapped events (NIP-59) for routing requests/responses through additional relays if metadata privacy is a high concern, though this adds complexity.

## Payment Receipt

To provide a definitive confirmation of payment, the receiving service MAY send a final `kind: 21001` event to the payer after the invoice has been successfully paid. This is distinct from a NIP-57 zap receipt and serves as a direct, private acknowledgment.

### Receipt Event

- **Kind**: `21001`
- **Sender**: Receiving Service
- **Recipient**: Payer
- **Tags**:
  - `["p", "<payer_pubkey>"]`
  - `["e", "<request_event_id>"]`
  - `["clink_version", "1"]`
- **Content**: NIP-44 encrypted JSON payload.

### Decrypted Receipt Payload

The event itself, being signed by the wallet service and referencing the original request via an `e` tag, serves as a verifiable acknowledgment. The payload distinguishes between a standard Lightning payment and an internal settlement.

1.  **Standard Lightning Payment:**
    The payload MUST include the `preimage` to prove the Lightning payment was settled.
    ```json
    {
      "res": "ok",
      "preimage": "<64-char_hex_lightning_preimage>"
    }
    ```

2.  **Internal Settlement:**
    For payments settled internally, the payload is a simple acknowledgment. The absence of a `preimage` indicates an internal transaction.
    ```json
    {
      "res": "ok"
    }
    ```

This flow provides a closed loop for programmatic interactions, allowing the payers application to verify that the payment was received and unlock content or services accordingly.

## Reference Implementations

- **Wallet Node:** [Lightning.Pub](https://github.com/shocknet/Lightning.Pub)
- **Wallet Client:** [ShockWallet](https://shockwallet.app)
- **SDK:** [CLINK SDK](https://github.com/shocknet/ClinkSDK)
- **Demo Client:** [clinkme.dev](https://clinkme.dev/) 