# CLINK: Common Lightning Interactions with Nostr Keys

CLINK defines Nostr-native standards for Lightning Network interactions, leveraging the protocol's built-in transport, identity, and encryption. This enables secure communication between Lightning nodes, wallets, and applications without traditional web infrastructure dependencies while also remaining web-friendly and facilitating self-custody. This novel use of Nostr delivers a greatly enhanced user experience and unlocks new use-cases versus the status quo of things like LNURL, Bolt12, and NWC.

- [About CLINK](about.md)
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

### CLINK Offers
- Lightning.Pub (reference implementation)
- ShockWallet
- *Your implementation here*

### CLINK Debits
- Lightning.Pub (reference implementation)
- ShockWallet
- *Your implementation here*

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
