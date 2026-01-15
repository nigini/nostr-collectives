# NIP-C: Commons Enforcement

**Status**: Draft
**Depends on**: NIP-A (Collective Identity), NIP-B (NosCAP)

## Summary

A **commons** is a named space within a collective where members can publish content. Each commons is an addressable event (kind:39002) that can have its own membership via caps. Relays validate caps before accepting events in commons they enforce.

## Motivation

Without relay enforcement:
- Anyone can claim to post in a collective's commons
- Caps are just metadata with no enforcement
- Collectives have no control over their spaces

Relay enforcement makes commons meaningful: only authorized members can publish.

## Specification

### Commons Definition (kind:39002)

A commons is defined as an addressable event:

```json
{
  "pubkey": "<collective_npub>",
  "kind": 39002,
  "tags": [
    ["d", "<uuid>"]
  ],
  "content": "{
    \"name\": \"Research Commons\",
    \"about\": \"Space for research discussions\",
    \"picture\": \"https://example.com/research.png\",
    \"relays\": [\"wss://relay.example.com\"]
  }",
  "created_at": 1704067200,
  "id": "...",
  "sig": "..."
}
```

**Fields in content (JSON)**:
| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Human-readable name |
| `about` | No | Description |
| `picture` | No | Avatar/icon URL |
| `relays` | No | Preferred relays for this commons |

**Why UUID for d-tag?**
- Guaranteed unique
- Rename-safe (name can change, ID stays)
- Simple to generate (`crypto.randomUUID()`)

### Multiple Commons per Collective

A collective can have multiple commons with different purposes:

```
Collective (npub1abc...)
├── Commons: "Research" (uuid: 550e8400-...)
│   └── Members: Alice, Bob, Carol
├── Commons: "Announcements" (uuid: 6ba7b810-...)
│   └── Members: Stewards only (read-only for others)
└── Commons: "General" (uuid: 7c9e6679-...)
    └── Members: All members
```

### Referencing Commons (NIP-33 `a` tag)

Events reference commons using the standard `a` tag for addressable events:

```json
{
  "pubkey": "<member_npub>",
  "kind": 1,
  "tags": [
    ["a", "39002:<collective_npub>:<uuid>"],
    ["cap", "<cap_reference>"]
  ],
  "content": "Content in the commons"
}
```

**Benefits of `a` tag**:
- Standard NIP-33 pattern
- Already indexed by relays
- No new infrastructure needed
- Works with existing filters: `#a`

### Ownership Rules

| Condition | Cap Required? |
|-----------|---------------|
| `pubkey == collective_npub` | No (collective posting directly) |
| `pubkey != collective_npub` | Yes (member needs cap) |

### Cap Attachment

Events must include cap reference:

**Option A: Event ID reference**
```json
["cap", "<cap_event_id>", "wss://relay.example.com"]
```

**Option B: Inline token**
```json
["cap", "noscap1<base64url_encoded_cap>"]
```

## Relay Configuration

Relays can enforce specific commons:

```json
{
  "enforced_commons": [
    {
      "commons": "39002:<collective_npub>:<uuid>",
      "require_cap": true,
      "allowed_kinds": [1, 30023, 30078]
    }
  ],
  "default_policy": "accept"
}
```

**Configuration options**:
- `commons` - The commons address (kind:pubkey:d-tag)
- `require_cap` - Whether caps are required
- `allowed_kinds` - Event kinds allowed in this commons
- `default_policy` - What to do for unconfigured commons

## Validation Flow

```
Client                              Relay
  │                                   │
  │  EVENT with a-tag + cap           │
  │──────────────────────────────────>│
  │                                   │
  │                    1. Is commons enforced?
  │                    2. Parse cap (inline or fetch)
  │                    3. Validate cap signature
  │                    4. Validate grantee matches pubkey
  │                    5. Validate action covers this kind
  │                    6. Validate commons matches cap
  │                    7. Check not expired
  │                    8. Check not revoked
  │                                   │
  │  OK / NOTICE "cap invalid: ..."   │
  │<──────────────────────────────────│
```

### Validation Steps

1. **Commons check**: Is this commons enforced by this relay?
2. **Pubkey check**: If pubkey == collective, accept (no cap needed)
3. **Cap parse**: Extract cap from tag (fetch if referenced)
4. **Signature check**: Was cap signed by the collective or valid delegator?
5. **Grantee check**: Does cap grantee match event pubkey?
6. **Action check**: Does cap authorize this kind?
7. **Commons check**: Does cap authorize this specific commons?
8. **Expiry check**: Is cap still valid (not expired)?
9. **Revocation check**: Has cap been revoked?

### Error Responses

```
NOTICE "cap required: commons 39002:<pubkey>:<uuid> is enforced"
NOTICE "cap invalid: signature verification failed"
NOTICE "cap invalid: grantee mismatch"
NOTICE "cap invalid: action not authorized for kind:1"
NOTICE "cap invalid: commons not authorized"
NOTICE "cap invalid: expired"
NOTICE "cap invalid: revoked"
```

## Querying Commons Content

Using standard NIP-33 `#a` filter (already supported by relays):

```json
// Query all events in a specific commons
{
  "kinds": [1],
  "#a": ["39002:<collective_npub>:<uuid>"]
}

// Query events in commons by specific author
{
  "kinds": [1],
  "#a": ["39002:<collective_npub>:<uuid>"],
  "authors": ["<member_npub>"]
}

// Query all commons definitions for a collective
{
  "kinds": [39002],
  "authors": ["<collective_npub>"]
}
```

## NIP-42 AUTH Extension

For private commons, extend AUTH with cap:

```json
// Standard NIP-42 challenge
["AUTH", "<challenge>"]

// Response with cap for commons read access
{
  "kind": 22242,
  "tags": [
    ["relay", "wss://relay.example.com"],
    ["challenge", "<challenge>"],
    ["cap", "<cap_reference>"]
  ]
}
```

This allows relays to verify read access for private commons.

## Performance Considerations

### Cap Caching

Relays should cache validated caps:
- Key: cap event ID
- Value: parsed cap + validation result
- TTL: min(cap expiry, 1 hour)

### Chain Depth Limits

For delegation chains, recommend max depth of 3-5 to bound validation time.

### Batch Validation

When processing multiple events from same member, reuse cap validation result.

## Relationship to Existing NIPs

| NIP | Relationship |
|-----|--------------|
| NIP-01 | Standard event structure |
| NIP-29 | Compatible - relay groups can adopt commons |
| NIP-33 | Uses addressable events pattern |
| NIP-42 | Extended for cap-based AUTH |
| NIP-65 | Collectives advertise supporting relays |
| NIP-72 | Communities can adopt commons |

## Open Questions

1. **Cross-relay validation**: Can relay A validate cap stored only on relay B?
2. **Performance**: What's the overhead for high-volume commons?
3. **Partial enforcement**: Should relays enforce some commons but not others?
4. **Gossip**: How do relays learn about revocations?
5. **Migration**: How to transition existing NIP-29 groups?
6. **Commons discovery**: Should commons definitions be published to relays or kept private?

## Event Kinds

| Kind | Description |
|------|-------------|
| 39002 | Commons definition |

## See Also

- [NIP-29: Relay-based Groups](https://github.com/nostr-protocol/nips/blob/master/29.md)
- [NIP-33: Parameterized Replaceable Events](https://github.com/nostr-protocol/nips/blob/master/33.md)
- [NIP-42: Authentication](https://github.com/nostr-protocol/nips/blob/master/42.md)
- [NIP-65: Relay List Metadata](https://github.com/nostr-protocol/nips/blob/master/65.md)
