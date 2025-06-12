# CLINK: Common Lightning Interactions with Nostr Keys

CLINK defines Nostr-native standards for Lightning Network interactions, leveraging the protocol's built-in transport, identity, and encryption. This enables secure communication between Lightning nodes, wallets, and applications without traditional web infrastructure dependencies while also remaining web-friendly and facilitating self-custody. This novel use of Nostr delivers a greatly enhanced user experience and unlocks new use-cases versus the status quo of things like LNURL, Bolt12, and NWC.

## About

### Purpose

The primary goal of CLINK is to provide standardized, **Nostr-native** methods for common Lightning interactions. By building directly on Nostr's capabilities, CLINK enables secure integration between Lightning applications and node services that enhances the user experience without compromising sovereignty.

### Motivation

Current Lightning interaction methods have a number of pitfalls. Some require maintaining traditional web infrastructure, adding undue complexity to self-hosting that results in custodial solutions. Others either introduce friction and security risks through pre-shared secrets, or suffer from dependence on slow and unreliable onion messages that are also impractical for use in web applications. These approaches have created significant barriers to broad adoption.

CLINK leverages Nostr's native strengths to enable direct, identity-driven interactions between Lightning nodes and applications. By building on Nostr's cryptographic primitives, CLINK aims to:

*   **Simplify User Experience:** Enable direct, spontaneous interactions in a way that is compatible with familiar Nostr identifiers (e.g., NIP-05 addresses).
*   **Reduce Infrastructure Dependency:** Operate entirely over the Nostr network where desired, eliminating the need for traditional web infrastructure.
*   **Enhance Security & Leverage Web-of-Trust:** Utilize Nostr's cryptographic identity and signed events for establishing connections and authorizing actions.
*   **Foster a Native Ecosystem:** Provide standardized protocols built *for* Nostr, enabling tighter integration between Lightning and Nostr-powered applications.

### Core CLINK Protocols

* **CLINK Offers:** Provides static payment codes (`noffer1...`) analogous to LNURL-Pay but entirely Nostr-native. Enables services to offer invoice generation via Nostr direct messages, removing the requirement for a publicly accessible HTTPS endpoint but without unreliable P2P performance. Services like Lightning.Pub can trigger webhooks on offer events, enabling easy integration with traditional application flows while maintaining self-custody.
* **CLINK Debits:** Provides static debit authorization pointers (`ndebit1...`) enabling direct, secure payment requests between parties by leveraging key-based identity and event-based authorization flows. This eliminates reliance on custodial APIs or key distribution for debit-style functionality.

### Is this similar to NWC?

While NWC also utilizes Nostr for transport, it specifically targets **wallet remote control** modeled after the RPC pioneered by Lightning.Pub and ShockWallet to provide a Nostr-tunneled API.

CLINK, conversely, defines **interactive protocols** using Nostr's wider range of capabilities such as identity and ad-hoc messaging that **enable direct application-service and peer-to-peer relationships**. This includes:
1.  **Service Interactions:** Providing specifications like `CLINK Offers` for invoice requests – functionality *not* covered by NWC.
2.  **Payment Workflows:** Defining protocols like `CLINK Debits` for event-driven payment requests and authorization.

This comprehensive and protocol-based approach facilitates a more integrated architecture for applications communicating directly over Nostr for diverse Lightning tasks. It enables enhanced security models tied to Nostr identity while maintaining compatibility with traditional web application architectures.

Where NWC is deferential to LNURL and scoped for a specific task, **CLINK is fundamentally committed to Nostr as the foundation for the next generation of decentralized Lightning applications.**

- [Specifications](#specifications)
- [Event Kinds](#event-kinds)
- [Implementation Status](#implementation-status)
- [Contributing](#contributing)

## Specifications

- [CLINK Offers](specs/clink-offers.md): Static payment codes (`noffer1...`) for invoice requests
- [CLINK Debits](specs/clink-debits.md): Static authorization pointers (`ndebit1...`) for payment requests

## Event Kinds

| kind | description | spec |
|------|-------------|------|
| `21001` | Offer Request/Response | [CLINK Offers](specs/clink-offers.md) |
| `21002` | Debit Request/Response | [CLINK Debits](specs/clink-debits.md) |

## Implementation Status

| Project      | Type    | Supports        | Features / Notes |
|--------------|---------|----------------|------------------|
| [Lightning.Pub](https://lightning.pub) | Server  | Offers, Debits | Reference server for wallets. |
| [ShockWallet](https://shockwallet.app) | Wallet  | Offers, Debits | UI for approval and rules for Lightning.Pub. |
| [Bridgelet](https://github.com/shocknet/bridgelet) | Bridge  | Offers         | NIP-05, LNURL, Lightning Address via CLINK Offers. |
| [CLINK SDK](https://www.npmjs.com/package/@shocknet/clink-sdk) | SDK     | Offers, Debits | JS/TS library for CLINK integration. |
| [clinkme.dev](https://clinkme.dev) | Client | Offers | Demo web-app for CLINK Offers. |
| *Your project here* | - | - | PR welcome! |

## Contributing

CLINK specifications aim to standardize common Lightning interactions over Nostr. To propose changes or additions:

1. **Implementation First**: New specifications should demonstrate working implementations.
2. **Backwards Compatible**: Changes must not break existing implementations.
3. **No Redundancy**: There should be no more than one way of doing the same thing.
4. **Nostr Native**: Specifications should leverage Nostr's inherent capabilities (identity, events, encryption).

### Process

1. **Discussion**: Open an issue to discuss the proposed change/addition
2. **Implementation**: Develop and test the feature
3. **Documentation**: Submit a PR with specification updates
4. **Review**: Community feedback and refinement
5. **Acceptance**: Merge when consensus is reached

## License

All CLINK specifications are public domain.
