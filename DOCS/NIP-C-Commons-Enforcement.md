# NIP-C: Commons Enforcement

- **Status**: Draft
- **Depends on**: [NIP-A (Collective Identity)](NIP-A-Collective-Identity.md), [NIP-B (NosCAP)](NIP-B-NosCAP.md)
- **Related**: [Overview](00-Overview.md), [Policy-Based Signer](Policy-Based-Signer.md)

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

```mermaid
flowchart TD
    C["Collective (npub1abc...)"] --> R["Research (550e8400...)"]
    C --> A["Announcements (6ba7b810...)"]
    C --> G["General (7c9e6679...)"]
    R --- R1["Members: Alice, Bob, Carol"]
    A --- A1["Members: Stewards only"]
    G --- G1["Members: All members"]
```

### Referencing Commons (NIP-33 `a` tag)

Events reference commons using the standard `a` tag for addressable events:

```json
{
  "pubkey": "<member_npub>",
  "kind": 1,
  "tags": [
    ["a", "39002:<collective_npub>:<uuid>"]
  ],
  "content": "Content in the commons"
}
```

**Benefits of `a` tag**:
- Standard NIP-33 pattern
- Already indexed by relays
- No new infrastructure needed
- Works with existing filters: `#a`

> **Note**: Events do NOT carry a CAP tag. Authorization is handled at the connection level via CAP-AUTH (see below), not per-event.

### Ownership Rules

| Condition | Cap Required? |
|-----------|---------------|
| `pubkey == collective_npub` | No (collective posting directly) |
| `pubkey != collective_npub` | Yes (member needs cap) |

## Commons Auto-Registration

When a relay receives a valid `kind:39002` event, it automatically registers that commons for enforcement. Any subsequent read/write to that commons requires appropriate CAP-AUTH.

```javascript
// Relay auto-registers commons on receipt
if (event.kind === 39002) {
  const uuid = event.tags.find(t => t[0] === "d")?.[1]
  enforcedCommons.add(`39002:${event.pubkey}:${uuid}`)
}
```

No static configuration file is needed — commons are enforced as soon as they are created.

## Enforcement

Authorization is checked at two points: **write** (EVENT) and **read** (REQ). Both use the grants stored during CAP-AUTH (see below).

### Write Enforcement (EVENT)

```mermaid
flowchart TD
    E["EVENT received"] --> A{"Has commons a-tag?"}
    A -->|No| ACCEPT["Accept"]
    A -->|Yes| B{"Commons enforced?"}
    B -->|No| ACCEPT
    B -->|Yes| C{"pubkey == collective?"}
    C -->|Yes| ACCEPT
    C -->|No| D{"Connection authenticated?"}
    D -->|No| DENY["Reject: auth-required"]
    D -->|Yes| F{"pubkey matches auth?"}
    F -->|No| DENY2["Reject: pubkey mismatch"]
    F -->|Yes| G{"Has publish grant for this commons?"}
    G -->|Yes| ACCEPT
    G -->|No| DENY3["Reject: unauthorized"]
```

### Read Enforcement (REQ)

For subscriptions, the relay silently filters events the connection cannot access:

- Events without a commons `a` tag: always included
- Events in non-enforced commons: always included
- Events in enforced commons: included only if the connection has `access` or `publish` grants for that commons

### Error Responses

Write rejections use the standard `OK` message with an error prefix:

```json
["OK", "<event_id>", false, "auth-required: this commons requires CAP authentication"]
["OK", "<event_id>", false, "restricted: no publish permission for this commons"]
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

## CAP-AUTH: Self-Contained Authorization

For both **read** (subscriptions) and **write** (publishing) access to protected commons, clients authenticate once per connection by presenting a **self-contained CAP-AUTH event**. This extends NIP-42 with embedded capability proof.

The client embeds the **full CAP** in a signed AUTH event. The relay validates both signatures in one step — no fetching, no external dependencies, everything needed is in the message.

### CAP-AUTH Event Structure

```json
{
  "kind": 22242,
  "pubkey": "<grantee_pubkey>",
  "created_at": 1704067200,
  "tags": [
    ["relay", "wss://relay.example.com"],
    ["challenge", "<relay_challenge>"],
    ["cap", "<full_cap_event_as_json>"]
  ],
  "content": "",
  "sig": "<grantee_signature>"
}
```

The `cap` tag contains the **full CAP event as JSON**, not just a reference:

```json
["cap", "{\"kind\":39100,\"pubkey\":\"<collective>\",\"tags\":[[\"p\",\"<grantee>\"],[\"cap\",\"publish\",\"*\"],[\"a\",\"39002:<collective>:*\"]],\"sig\":\"...\"}"]
```

### Two Signatures, One Message

A CAP-AUTH event carries two nested signatures that together prove both **identity** (who is making the request) and **authorization** (what they're allowed to do and where). The relay validates both in a single step with no external lookups:

```mermaid
flowchart TB
    subgraph AUTH["CAP-AUTH Event (signed by grantee)"]
        A1["pubkey: grantee — WHO is requesting"]
        A2["sig: grantee — PROOF of identity"]
        subgraph CAP["Embedded CAP (signed by collective)"]
            C1["pubkey: collective — WHO authorized"]
            C2["tag p: grantee — WHO is authorized"]
            C3["tag cap: publish, * — WHAT is allowed"]
            C4["tag a: 39002:xyz:uuid — WHERE it applies"]
            C5["sig: collective — PROOF of grant"]
        end
    end

    style AUTH fill:#e0e7ff,stroke:#6366f1
    style CAP fill:#ddd6fe,stroke:#8b5cf6
```

### Connection State After AUTH

Once validated, the relay stores the connection's grants as a flat list:

```javascript
connections.set(socketId, {
  pubkey: "grantee_abc123",
  grants: [
    { action: "publish", scope: "*", commons: "39002:collective_xyz:uuid" },
    { action: "access", scope: "*", commons: "39002:collective_xyz:uuid" }
  ]
})
```

Each grant has an `action` (publish, access), a `scope` (event kind or `*` for all), and the `commons` reference it applies to.

### Full Flow

```mermaid
sequenceDiagram
    participant C as Client
    participant R as Relay

    C->>R: CONNECT (WebSocket)
    R->>C: AUTH challenge

    C->>R: CAP-AUTH (signed, with full CAP)

    Note right of R: 1. Verify outer sig
    Note right of R: 2. Parse embedded CAP
    Note right of R: 3. Verify CAP sig (kind:39100)
    Note right of R: 4. Check grantee match
    Note right of R: 5. Check expiry
    Note right of R: 6. Store grants

    R->>C: OK (authenticated)

    C->>R: REQ / EVENT
    Note right of R: Enforced per connection grants
```

## Relationship to Existing NIPs

| NIP | Relationship |
|-----|--------------|
| NIP-01 | Standard event structure |
| NIP-11 | Relay information document (relay should advertise NIP-42 support) |
| NIP-29 | Compatible - relay groups can adopt commons |
| NIP-33 | Uses addressable events pattern |
| NIP-42 | Extended for CAP-based AUTH |
| NIP-65 | Collectives advertise supporting relays |

## Open Questions

1. **Revocation**: The current design embeds the full CAP in CAP-AUTH, making validation self-contained (no external lookups). But this means a revoked CAP remains valid until it expires. Two approaches under consideration:
   - **Expiry-based** (current): CAPs have short TTL and must be renewed. Revocation = don't renew. Simple but coarse — revocation takes effect only after expiry.
   - **Replaceable CAPs**: Make CAPs replaceable events (kind:39100 with a `d` tag per grantee). The issuer can update or blank the CAP to revoke it. The relay fetches the latest CAP version at AUTH time instead of trusting the embedded copy. This adds latency and an external dependency, but provides real-time revocation. A relay could cache CAPs and refresh periodically to balance speed and freshness.
2. **Migration**: How to transition existing NIP-29 groups?
3. **Commons discovery**: Should commons definitions be published to relays or kept private?

## Future Work

- **CAP revocation** (`kind:39101`): Relays should check revocation status during AUTH validation
- **Delegation chains**: Validate `parent` tag chains with depth limits (3-5 max)
- **CAP kind validation**: Verify embedded CAP is `kind:39100` (not just any signed event)
- **`relay` tag validation**: Check AUTH event `relay` tag matches to prevent cross-relay replay
- **`delete` action enforcement**: Separate grant for `kind:5` deletion events

## Event Kinds

| Kind | Description |
|------|-------------|
| 39002 | Commons definition |
| 22242 | CAP-AUTH event (NIP-42 extension) |

## See Also

- [NIP-29: Relay-based Groups](https://github.com/nostr-protocol/nips/blob/master/29.md)
- [NIP-33: Parameterized Replaceable Events](https://github.com/nostr-protocol/nips/blob/master/33.md)
- [NIP-42: Authentication](https://github.com/nostr-protocol/nips/blob/master/42.md)
- [NIP-65: Relay List Metadata](https://github.com/nostr-protocol/nips/blob/master/65.md)
