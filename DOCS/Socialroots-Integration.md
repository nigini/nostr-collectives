# Socialroots Integration

## About Socialroots

[Socialroots](https://socialroots.io) is a platform for self-governed online communities. It explores how groups can have:

- **Autonomy**: Self-governance, own rules and policies
- **Portability**: Not locked to any single platform
- **Network power**: Form meaningful relationships with other groups

Socialroots provides the inspiration for Nostr Collectives and serves as a primary use case for this proposal.

## The Decentralization Use Case

### Current Architecture

Socialroots uses a microservice architecture with protocol gateways:

```
┌─────────────────────────────────────────────────────────┐
│                    SOCIALROOTS CORE                      │
│                                                          │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐        │
│  │ RS-USERS   │  │ RS-GROUPS  │  │ RS-NOTES   │        │
│  └────────────┘  └────────────┘  └────────────┘        │
│         │              │              │                 │
│         └──────────────┼──────────────┘                 │
│                        │                                │
│                 ┌──────┴──────┐                         │
│                 │ RS-MEMBRANE │                         │
│                 │ (identity)  │                         │
│                 └──────┬──────┘                         │
│                        │                                │
└────────────────────────┼────────────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
    ┌────┴────┐    ┌────┴────┐    ┌────┴────┐
    │  NOSTR  │    │  EMAIL  │    │ MATRIX  │
    │ GATEWAY │    │ GATEWAY │    │ GATEWAY │
    └─────────┘    └─────────┘    └─────────┘
```

### The Problem: Gateway Centralization

Currently, the **SR-NOSTR-GATEWAY**:
- Holds a single nsec for all Nostr interactions
- Signs events on behalf of the platform, not groups
- Creates a centralization point

**Result**: Socialroots groups don't have Nostr identity—the platform does.

### The Solution: Collective Identity per Group

With Nostr Collectives:

```
┌─────────────────────────────────────────────────────────┐
│                    SOCIALROOTS CORE                      │
│                                                          │
│  ┌────────────┐                                         │
│  │ RS-GROUPS  │ ◄── Each group gets a collective npub   │
│  └─────┬──────┘                                         │
│        │                                                │
│  ┌─────┴──────┐                                         │
│  │ RS-MEMBRANE│ ◄── Coordinates identity + caps         │
│  └─────┬──────┘                                         │
│        │                                                │
└────────┼────────────────────────────────────────────────┘
         │
    ┌────┴────────────────────────┐
    │       SR-NOSTR-GATEWAY      │
    │                             │
    │  ┌─────────┐  ┌─────────┐  │
    │  │Group A  │  │Group B  │  │  ◄── Per-group npub/nsec
    │  │ npub    │  │ npub    │  │
    │  └─────────┘  └─────────┘  │
    │                             │
    │  Issues caps to members     │
    └─────────────────────────────┘
```

**Result**: Each Socialroots group IS a Nostr collective with its own identity.

## Integration Points

### RS-MEMBRANE (Identity Coordinator)

**Current role**:
- Stores DID:KEY identities for users
- Issues UCANs for internal permissions
- Manages entity relationships

**Extended role**:
- Store Nostr npub as additional identity type
- Coordinate cap issuance across gateways
- Map internal permissions to Nostr caps

```rust
// Entity with Nostr identity
Entity {
    id: UUID,
    did_key: "did:key:z6Mk...",
    nostr_npub: "npub1...",     // NEW
    metadata: {...}
}
```

### SR-NOSTR-GATEWAY (Protocol Gateway)

**Current role**:
- Single nsec for platform
- Sign and publish events
- Relay management

**Extended role**:
- Store npub/nsec per collective
- Issue kind:39100 caps for members
- Sign events with collective's key

```rust
// Per-collective key storage
CollectiveKeys {
    group_id: UUID,
    npub: "npub1...",
    nsec: Encrypted<"nsec1...">,  // HSM or bunker
    stewards: Vec<Pubkey>
}
```

### RS-GROUPS (Group Service)

**Current role**:
- UUID-based group identification
- Role-based membership
- Group settings and policies

**Extended role**:
- Reference collective npub
- Map roles to cap templates
- Track collective metadata

```rust
// Group with collective identity
Group {
    id: UUID,
    name: "Climate Action Network",
    collective_npub: Option<Pubkey>,  // NEW
    membership_model: CapBased,        // NEW
    role_caps: HashMap<Role, CapTemplate>  // NEW
}
```

## Migration Path

### Phase 1: Opt-in Collective Identity

1. Group admin enables "Nostr Collective" feature
2. Gateway generates npub/nsec for group
3. Gateway publishes kind:0 profile
4. Members can view collective npub

### Phase 2: Cap-Based Membership

1. When user joins group, gateway issues cap
2. Cap grants publish rights per role
3. Members can post to collective's commons
4. Gateway or member signs events

### Phase 3: Portable Identity

1. Members export caps for offline use
2. Members can post directly to Nostr relays
3. Collective identity works outside Socialroots

### Phase 4: Inter-Collective Networks

1. Groups exchange caps for federation
2. Cross-group access via cap delegation
3. Socialroots becomes one node in larger network

## Benefits for Socialroots

| Benefit | Description |
|---------|-------------|
| **Decentralization** | No single point of failure for group identity |
| **Portability** | Groups survive if Socialroots goes down |
| **Interoperability** | Groups connect with non-Socialroots collectives |
| **User ownership** | Members own their relationship to group |
| **Network effects** | Groups benefit from broader Nostr ecosystem |

## Benefits for Nostr

| Benefit | Description |
|---------|-------------|
| **Real-world use case** | Socialroots brings organized groups to Nostr |
| **Battle testing** | Production usage reveals edge cases |
| **Libraries** | Socialroots contributes Go/JS implementations |
| **Documentation** | Integration guides from real experience |

## Technical Considerations

### Key Security

- Per-collective keys stored in HSM or NIP-46 bunker
- Steward authentication via Socialroots auth
- Key rotation without changing npub (future work)

### Cap Lifecycle

```
User joins group
    │
    ▼
RS-MEMBRANE notifies SR-NOSTR-GATEWAY
    │
    ▼
Gateway issues kind:39100 cap
    │
    ▼
Cap published to configured relays
    │
    ▼
User can now post to collective's commons
```

### Event Signing

Two modes:
1. **Gateway signs**: User submits content, gateway signs with collective key
2. **User signs**: User has cap, signs own events with personal key

Mode 2 is more decentralized but requires client changes.

## Open Questions

1. **Key custody**: Who holds collective nsec long-term?
2. **Steward mapping**: How do Socialroots admins become Nostr stewards?
3. **Cap sync**: How to keep caps in sync across Socialroots and Nostr?
4. **Conflict resolution**: What if cap state diverges?
5. **Exit strategy**: How do groups migrate away from Socialroots?

## Prototype Plan

### Milestone 1: Proof of Concept
- [ ] Create one collective npub manually
- [ ] Issue one cap to a test user
- [ ] Post one event to collective's commons
- [ ] Verify with standard Nostr client

### Milestone 2: Gateway Integration
- [ ] Add per-group key storage to gateway
- [ ] Implement cap issuance on membership change
- [ ] Publish events with collective identity

### Milestone 3: Full Integration
- [ ] RS-MEMBRANE identity linking
- [ ] RS-GROUPS collective settings UI
- [ ] Member cap management
- [ ] Relay enforcement testing

## See Also

- [Nostr Collectives Overview](00-Overview.md)
- [NIP-A: Collective Identity](NIP-A-Collective-Identity.md)
- [NIP-B: NosCAP](NIP-B-NosCAP.md)
