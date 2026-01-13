# Nostr Collectives

**Shared Identity and Capabilities for Nostr**

> What if groups could exist as first-class citizens on Nostr?

## The Idea

A **collective** is an npub that represents a group—not an individual. Members receive **capabilities (caps)** to act within the collective's **space**. Stewards govern who can do what.

```
Today:    Identity = Individual (one person, one npub)
Proposed: Identity = Individual OR Collective
```

This enables:
- **Shared identity**: A collective owns an npub, profile, and voice
- **Delegated action**: Members publish on behalf of the collective via caps
- **Collective networks**: Groups form alliances by exchanging caps with each other

## Three Primitives

| Primitive | Description | NIP |
|-----------|-------------|-----|
| **Collective Identity** | A group as an npub with stewards | [NIP-A](DOCS/NIP-A-Collective-Identity.md) |
| **NosCAP** | Capability grants for actions in a space | [NIP-B](DOCS/NIP-B-NosCAP.md) |
| **Space Enforcement** | Relay validation of caps | [NIP-C](DOCS/NIP-C-Space-Enforcement.md) |

## Documentation

- [Overview](DOCS/00-Overview.md) - Core concepts and terminology
- [NIP-A: Collective Identity](DOCS/NIP-A-Collective-Identity.md) - What a collective IS
- [NIP-B: NosCAP](DOCS/NIP-B-NosCAP.md) - Capability system
- [NIP-C: Space Enforcement](DOCS/NIP-C-Space-Enforcement.md) - Relay validation
- [Socialroots Integration](DOCS/Socialroots-Integration.md) - Use case and architecture

## Roadmap

### Phase 0: Prototype (Month 1)
- [ ] Create a collective npub (single custodian)
- [ ] Issue one cap to a member
- [ ] Member publishes to collective's space
- [ ] Existing client displays post "from" collective

### Phase 1: Foundation (Months 2-4)
- [ ] NIP drafts: Collective Identity, NosCAP
- [ ] Go + JS libraries for cap issuance/validation
- [ ] Test suite for cap validation scenarios

### Phase 2: Relay Support (Months 5-7)
- [ ] NIP draft: Space Enforcement
- [ ] Relay patch (strfry or nostr-rs-relay)
- [ ] Performance testing

### Phase 3: Collective Tools (Months 8-10)
- [ ] Collective creation workflow
- [ ] Membership UI
- [ ] NIP-46 bunker integration
- [ ] Demo: two collectives forming a network

### Phase 4: Ecosystem Integration (Months 11-12)
- [ ] Client integration guide
- [ ] Frostr integration for threshold signing
- [ ] Documentation and tutorials

## Inspiration

This project is inspired by [Socialroots](https://socialroots.io), a platform for self-governed online communities. Socialroots explores how groups can have **autonomy** (self-governance), **portability** (not locked to platforms), and **network power** (form relationships with other groups).

The Nostr Collectives proposal brings these ideas to the Nostr protocol, enabling decentralized collective identity that works with existing clients and relays.

See [Socialroots Integration](DOCS/Socialroots-Integration.md) for how Socialroots plans to use Nostr Collectives to decentralize its infrastructure.

## Status

**Draft** - This is an early-stage proposal. Feedback welcome!

