# About CLINK: Common Lightning Interactions with Nostr Keys

**CLINK** stands for **Common Lightning Interactions with Nostr Keys**. It is a suite of lightweight communication protocols designed to enable secure and direct interaction between Lightning nodes and applications using the Nostr network for transport, identity, and encryption.

### Purpose

The primary goal of CLINK is to provide standardized, **Nostr-native** methods for common Lightning interactions. By building directly on Nostr's capabilities, CLINK enables seamless integration between Lightning applications and node services without introducing additional dependencies.

### Motivation

Current Lightning interaction methods face significant limitations. Some require maintaining traditional web infrastructure, adding undue complexity to self-hosting that results in custodial solutions. Others either introduce friction and security risks through pre-shared secrets, or suffer from dependence on slow and unreliable onion messages that are also impractical for use in web applications. These approaches have created significant barriers to broad adoption.

CLINK leverages Nostr's native strengths to enable direct, identity-driven interactions between Lightning nodes and applications. By building on Nostr's cryptographic primitives, CLINK aims to:

*   **Simplify User Experience:** Enable direct, spontaneous interactions in a way that is compatible with familiar Nostr identifiers (e.g., NIP-05 addresses).
*   **Reduce Infrastructure Dependency:** Operate entirely over the Nostr network where desired, eliminating the need for traditional web infrastructure.
*   **Enhance Security & Leverage Web-of-Trust:** Utilize Nostr's cryptographic identity and signed events for establishing connections and authorizing actions.
*   **Foster a Native Ecosystem:** Provide standardized protocols built *for* Nostr, enabling tighter integration between Lightning and Nostr-powered applications.

### Core CLINK Protocols

* **CLINK Offers:** Provides static payment codes (`noffer1...`) analogous to LNURL-Pay but entirely Nostr-native. Enables services to offer invoice generation via Nostr direct messages, removing the requirement for a publicly accessible HTTPS endpoint but without flakey P2P performance. Services like Lightning.Pub can trigger webhooks on offer events, enabling easy integration with traditional application flows while maintaining self-custody.
* **CLINK Debits:** Provides static debit authorization pointers (`ndebit1...`) enabling direct, secure payment requests between parties by leveraging key-based identity and event-based authorization flows. This eliminates reliance on custodial APIs or insecure key fumbling for debit-style functionality.

### Isn't this similar to NWC?

While NWC also utilizes Nostr for transport, it specifically targets **wallet remote control** modeled after the RPC created pioneered by Lightning.Pub and ShockWallet to provide a Nostr-tunneled API in situations where the user-administrator needs multiple points of access.

CLINK, conversely, defines **interactive protocols** using core Nostr capabilities (identity, events, encryption) for a **range of direct application-service and peer-to-peer relationships**. This includes:
1.  **Service Interactions:** Providing specifications like `CLINK Offers` for invoice requests – functionality *not* covered by NWC.
2.  **Payment Workflows:** Defining protocols like `CLINK Debits` for event-driven payment requests and authorization flows.

This comprehensive, protocol-based approach facilitates a more integrated architecture for applications communicating directly over Nostr for diverse Lightning tasks. It enables enhanced security models tied to Nostr identity while maintaining compatibility with traditional web application architectures.

Where NWC was designed deferentially to LNURL for a specific task, **CLINK is fundamentally committed to Nostr as the comprehensive foundation for the next generation of decentralized Lightning applications.**