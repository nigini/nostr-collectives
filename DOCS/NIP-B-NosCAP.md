# NIP-B: NosCAP (Nostr Capabilities)

**Status**: Draft
**Depends on**: NIP-A (Collective Identity)
**Required by**: NIP-C (Commons Enforcement)

## Summary

A **cap** (capability) is a Nostr event granting specific actions to a member. It's the proof that someone has permission to act within a collective's commons.

Caps can be signed by:
1. **The collective** (npub) - most authoritative, direct from collective's nsec
2. **A steward** - common case, on behalf of the collective
3. **Any cap holder with `delegate` rights** - for delegation chains with attenuation

## Motivation

Current Nostr delegation (NIP-26) is limited:
- Binary (full delegation or nothing)
- No action scoping
- No attenuation (re-delegation with fewer rights)

NosCAP provides fine-grained, attenuatable capabilities inspired by UCAN and object-capability security.

## Specification

### Cap Event (kind:39100)

```json
{
  "pubkey": "<issuer_npub>",
  "kind": 39100,
  "tags": [
    ["p", "<grantee_npub>"],
    ["cap", "publish", "kind:1"],
    ["cap", "publish", "kind:30078"],
    ["cap", "delete", "kind:1"],
    ["a", "39002:<collective_npub>:<commons_uuid>"],
    ["expiry", "1767225600"],
    ["parent", "<parent_cap_event_id>"]
  ],
  "content": "",
  "created_at": 1704067200,
  "id": "...",
  "sig": "..."
}
```

### Tags

| Tag | Required | Description |
|-----|----------|-------------|
| `p` | Yes | Grantee pubkey |
| `cap` | Yes (1+) | Action grant: `[action, scope]` |
| `a` | Yes | Commons reference (NIP-33 addressable event: `39002:pubkey:d-tag`) |
| `expiry` | No | Unix timestamp when cap expires |
| `parent` | No | Parent cap event ID (for delegation chains) |

### Actions

| Action | Description |
|--------|-------------|
| `publish` | Create events of specified kinds |
| `delete` | Delete events (kind:5) |
| `access` | Read private commons content (future) |
| `delegate` | Issue attenuated caps to others |

### Scope Syntax

```
kind:1              → Specific kind
kind:30078          → Another specific kind
kind:30078:*        → Parameterized kind (any d-tag)
*                   → All kinds (steward-level)
```

**Examples**:
```json
["cap", "publish", "kind:1"]           // Can publish kind:1 (notes)
["cap", "publish", "kind:30023"]       // Can publish kind:30023 (long-form)
["cap", "publish", "*"]                // Can publish any kind
["cap", "delete", "kind:1"]            // Can delete kind:1 events
["cap", "delegate", "kind:1"]          // Can delegate kind:1 publish cap
```

## Cap Encoding

Caps can be transmitted in two ways when attached to events.

### Option A: Referenced (Event ID)

Event includes cap event ID + relay hints:

```json
{
  "pubkey": "<member_npub>",
  "kind": 1,
  "tags": [
    ["a", "39002:<collective_npub>:<commons_uuid>"],
    ["cap", "<cap_event_id>", "wss://relay.example.com"]
  ],
  "content": "Hello from the collective!"
}
```

| Pros | Cons |
|------|------|
| Smaller event payload | Relay must fetch cap separately |
| Cap reusable across events | Extra round-trip latency |
| Standard Nostr event reference | Dependency on cap-hosting relay |

### Option B: Inline (NosCAP Token)

Cap encoded as base64url token inline:

```json
{
  "pubkey": "<member_npub>",
  "kind": 1,
  "tags": [
    ["a", "39002:<collective_npub>:<commons_uuid>"],
    ["cap", "noscap1<base64url_encoded_cap>"]
  ],
  "content": "Hello from the collective!"
}
```

**Token format** (inspired by UCAN JWT):
```
noscap1<base64url([version, issuer, grantee, caps[], commons, expiry, created_at, sig])>
```

| Pros | Cons |
|------|------|
| Self-contained, no extra fetches | Larger event payload |
| Works offline / across relays | Cap repeated in every event |
| Faster validation | More complex parsing |

### Recommendation

Support both formats:
- **Referenced**: For long-lived memberships, routine use
- **Inline**: For cross-relay scenarios, ephemeral access, offline validation

## Delegation Chains

Caps can be re-delegated with attenuation (fewer rights).

```
Collective ──cap──► Steward ──cap──► Contributor
   (*)              (publish)         (publish:kind:1)
```

### Chain Structure

```json
// Steward's cap (from collective) - grants access to ALL commons
{
  "pubkey": "<collective_npub>",
  "kind": 39100,
  "tags": [
    ["p", "<steward_npub>"],
    ["cap", "publish", "*"],
    ["cap", "delegate", "*"],
    ["a", "39002:<collective_npub>:*"]
  ]
}

// Steward's cap (from collective) - grants access to specific commons
{
  "pubkey": "<collective_npub>",
  "kind": 39100,
  "tags": [
    ["p", "<steward_npub>"],
    ["cap", "publish", "*"],
    ["cap", "delegate", "*"],
    ["a", "39002:<collective_npub>:550e8400-e29b-41d4-a716-446655440000"]
  ]
}

// Contributor's cap (from steward) - attenuated to kind:1 in specific commons
{
  "pubkey": "<steward_npub>",
  "kind": 39100,
  "tags": [
    ["p", "<contributor_npub>"],
    ["cap", "publish", "kind:1"],
    ["a", "39002:<collective_npub>:550e8400-e29b-41d4-a716-446655440000"],
    ["parent", "<steward_cap_event_id>"]
  ]
}
```

### Validation Rules

1. **Signature valid**: Issuer signed the cap
2. **Grantee matches**: Event pubkey matches cap grantee
3. **Action covered**: Event kind included in cap scope
4. **Not expired**: Current time < expiry
5. **Chain valid**: If parent exists, recursively validate
6. **Attenuation respected**: Child caps ⊆ parent caps

## Revocation

### Expiration-Based

Caps have short TTL and must be renewed:

```json
["expiry", "1704153600"]  // Expires in 24 hours
```

### Explicit Revocation (kind:39101)

Steward publishes revocation event:

```json
{
  "pubkey": "<steward_npub>",
  "kind": 39101,
  "tags": [
    ["e", "<cap_event_id>"],
    ["p", "<revoked_member_npub>"]
  ],
  "content": "Membership ended",
  "created_at": 1704067200
}
```

### Emergency Blocklist

For compromised caps, relays may support npub-level blocks as an override mechanism. This is relay-specific policy, not protocol-level.

## Learning from UCAN

| UCAN Concept | NosCAP Equivalent |
|--------------|-------------------|
| Resource URI | `["a", "39002:collective_npub:commons_uuid"]` (NIP-33 addressable reference) |
| Capability | `["cap", "action", "scope"]` |
| Attenuation | Child cap ⊆ parent cap |
| Proof chain | `["parent", "cap_event_id"]` |
| Expiration | `["expiry", "timestamp"]` |
| Issuer/Audience | `pubkey` / `["p", "grantee"]` |

**Why not pure UCAN?**
- UCAN uses JWT format (foreign to Nostr ecosystem)
- UCAN assumes Ed25519 (Nostr uses secp256k1)
- Nostr events are self-contained and relay-native
- Cap-as-event enables discovery and relay caching

## Open Questions

1. **Cap storage**: Published to relays (discoverable) or exchanged via NIP-44 DMs (private)?
2. **Delegation depth**: Limit chain length for validation performance?
3. **Cap renewal**: Auto-renewal flow or manual?
4. **Partial revocation**: Revoke specific actions without full revocation?
5. **Collective as CAP-only issuer**: Should collectives be restricted to signing only CAPs (and metadata), with stewards always signing content as themselves? This improves accountability and enables cold storage of collective keys.
6. **Specialized CAP bunker**: If collectives are CAP-only issuers, should the collective's NIP-46 bunker be specialized for CAP-issuing rather than general signing? Trade-off: simpler bunker interface vs. reusability of CAP-issuing logic across implementations.

## Event Kinds

| Kind | Description |
|------|-------------|
| 39100 | Capability grant (cap) |
| 39101 | Capability revocation |

## See Also

- [NIP-26: Delegated Event Signing](https://github.com/nostr-protocol/nips/blob/master/26.md)
- [UCAN Specification](https://github.com/ucan-wg/spec)
- [Object-Capability Security](https://en.wikipedia.org/wiki/Object-capability_model)
