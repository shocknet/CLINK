# CLINK Offers Specification

## Overview

This specification defines **CLINK Offers**, a format for static payment codes in Nostr (`noffer1...`). It serves as a Nostr-native successor to LNURL-Pay, enabling users to initiate Lightning Network payments by scanning or clicking a string, without relying on traditional web servers.

## Motivation

Current Lightning payment flows either require maintaining HTTP endpoints, leading to unnecessary complexity and centralization risks in self-hosted scenarios, or depend on slow and unreliable transport mechanisms that are impractical for web applications. By leveraging Nostr's native capabilities for messaging and encryption, CLINK Offers provides a more direct and robust alternative that eliminates these dependencies.

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
- `4`: (Optional) The price in millisatoshis (integer) primarily for display purposes or fixed price offers.
- `5`: (Optional) Currency code (e.g., "USD", "EUR") if the price in TLV 4 represents a non-satoshi amount. If present, pricing type `1` (Variable) MUST be used.

**Default Behavior:** If neither price (TLV `4`) nor pricing type (TLV `3`) is present, the offer SHOULD be treated as type `2` (Spontaneous payment).

**Amount Unit:** Amounts in TLV `4` and request/response payloads are specified in **millisatoshis** for consistency with Lightning invoices and related NIPs (NIP-57, NIP-47), even though settlement typically happens at the satoshi level.

**Example Structure:**
```
noffer1...
  0: <receiver_public_key_hex>
  1: <relay_url>
  2: <offer_id_string>
  3: <pricing_type_flag> (0, 1, or 2, optional)
  4: <price_in_msats> (integer, optional)
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
*(Note: The exact field name, like `clink_offer` or `nip69` if backward compatibility is desired, should be finalized)*

### NIP-05 Lookup ("Lightning Addresses")

To support trust-minimized Lightning Address functionality, NIP-05 servers can include a mapping for offers (typically spontaneous payment offers) using a field like `nip69` or a CLINK-specific name.

**Example NIP-05 Response:**
```json
{
  "names": {
    "bob": "<hex_pub>"
  },
  "nip69": { // Or potentially clink_offer
    "bob": "noffer1..."
  }
}
```

### NIP-57 Zaps

CLINK Offers can replace the LNURL-pay callback step in the NIP-57 zap flow:

1.  The zapping client creates a kind `9734` zap request event.
2.  Instead of hitting an LNURL-P endpoint, the client sends a CLINK Offer payment request (Kind `21001`, see below) to the recipient's service pubkey found in the `noffer`.
3.  The encrypted payload of the Kind `21001` request includes both the `offer` identifier and the full kind `9734` event as a stringified JSON object in a `zap_request` field.
4.  The receiving service decrypts the Kind `21001` content, extracts the kind `9734` event, validates it, and uses its details (amount, tags) to generate the invoice.
5.  The receiver responds with the invoice via a Kind `21001` response event.
6.  Once the payer pays the invoice, the receiver generates and publishes the kind `9735` zap receipt as specified in NIP-57.

**Recommendation for Zap Offers:** Services supporting zaps SHOULD use an offer identifier starting with `zap` (e.g., `zap_default`, `zap_profile`) in their `noffer` string. Clients seeing this prefix can be confident in sending the `zap_request` payload. Services receiving a request for a zap-prefixed offer *without* a `zap_request` payload SHOULD treat it as a standard spontaneous payment.

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
    ["p", "<receiver_service_pubkey_hex>"]
  ],
  "content": "<NIP-44 encrypted request payload>",
  "sig": "<signature>"
}
```

**Decrypted Request Payload (`content`)**: A JSON object with the following fields:

```json
{
    "offer": "<offer_id_string>", // From noffer TLV 2
    "amount_msats": <amount_in_msats_integer>, // Required for spontaneous/variable, optional otherwise
    "payer_data": { ... }, // Optional: Arbitrary JSON object with payer info (e.g., NIP-05, name, pubkey)
    "zap_request": "{...}" // Optional: Stringified JSON of kind 9734 zap request event for NIP-57 flow
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
        ["e", "<request_event_id>"]
      ],
      "content": "<NIP-44 encrypted {\"res\":\"ok\",\"bolt11\":\"<BOLT11_invoice_string>\"}>",
      "sig": "<signature>"
    }
    ```

2.  **Error Response:**
    ```json
    {
      // ... similar structure ...
      "content": "<NIP-44 encrypted {\"res\":\"error\",\"reason\":\"<human_readable_error>\"}>",
      "sig": "<signature>"
    }
    ```
    *Common reasons might include: invalid offer ID, invalid amount for offer type, rate limit exceeded, internal error.*

## General Process Flow

1.  **Initiation**: Payer scans/clicks `noffer` string or uses NIP-05 lookup.
2.  **Decoding**: Payer's wallet decodes the `noffer` to get service pubkey, relay hint, offer ID, and pricing info.
3.  **Request**: Payer's wallet constructs and sends a Kind `21001` request event to the relay, addressed to the service pubkey.
    *   Includes `offer` ID.
    *   Includes `amount_msats` if offer type is spontaneous (`2`) or variable (`1`).
    *   Optionally includes `payer_data`.
    *   Optionally includes `zap_request` payload if performing a NIP-57 zap.
4.  **Service Processing**: Receiving service listens for Kind `21001` events.
    *   Decrypts payload.
    *   Validates the `offer` ID and `amount_msats` against offer parameters.
    *   (If variable price) Calculates the current price in msats.
    *   (If zap) Processes the `zap_request` event.
    *   Generates a BOLT11 Lightning invoice.
5.  **Response**: Service sends a Kind `21001` response event back to the payer's pubkey.
    *   Success: Encrypted payload contains `{"res":"ok", "bolt11":"..."}`.
    *   Failure: Encrypted payload contains `{"res":"error", "reason":"..."}`.
6.  **Payment**: Payer's wallet receives the response, decrypts it.
    *   If `ok`, presents the invoice to the user for payment (or pays automatically via NWC/CLINK Debits etc.).
    *   If `error`, displays the reason to the user.
7.  **Receipt (Optional)**: Post-payment receipt handling is outside the scope of this spec but could involve: NIP-57 zap receipts, Lightning pre-image sharing, or other out-of-band methods.

## Implementation Guidance

### Payer Wallet
- MUST support decoding `noffer` strings.
- MUST support sending Kind `21001` requests with NIP-44 encryption.
- MUST support receiving Kind `21001` responses and decrypting them.
- SHOULD handle different offer types (fixed, variable, spontaneous).
- SHOULD support NIP-57 zap flow integration by including the `zap_request` payload.
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