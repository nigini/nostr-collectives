# Nostr Collectives: Overview

## The Problem

### Today's Nostr Assumption

```
Identity = Individual
One person, one npub, one voice.
```

Groups in Nostr (NIP-29, NIP-72) are **containers**, not **actors**. They can't:
- Speak with a collective voice
- Own portable identity across relays
- Form relationships with other groups

### The Broader Problem (Beyond Nostr)

This isn't just a Nostr issue—it's the fundamental architecture of online groups:

| Dimension | Current State | Pain |
|-----------|---------------|------|
| **Identity** | Group exists "inside" a platform | No agency, no voice outside the silo |
| **Membership** | Platform-specific, non-portable | Rebuild reputation from scratch everywhere |
| **Relationships** | Groups are isolated islands | Can't federate, ally, or recognize each other |
| **Continuity** | Platform owns the group | Deplatforming = death, no migration path |
| **Governance** | Platform dictates rules | No self-determination |

> **Groups are tenants, not citizens.**

## The Solution

### Collective Identity

```
Identity = Individual OR Collective
A collective is an npub that represents shared identity.
Members act within the collective's commons via capabilities.
Stewards govern who can do what.
```

### Three Primitives

```
┌─────────────────────────────────────────────────────────────┐
│                   COLLECTIVE IDENTITY                        │
│              (npub that represents a group)                  │
│                                                              │
│   ┌─────────────────────┐    ┌─────────────────────┐        │
│   │       NosCAP        │    │  Commons Enforcement│        │
│   │   (capabilities)    │───►│  (relay validation) │        │
│   └─────────────────────┘    └─────────────────────┘        │
└─────────────────────────────────────────────────────────────┘
```

| Primitive | What it does |
|-----------|--------------|
| **Collective Identity** | Defines what a collective IS—an npub with stewards |
| **NosCAP** | How capabilities are granted and attenuated |
| **Commons Enforcement** | How relays validate caps and index collective content |

## Core Concepts

### Collective

An **npub** that represents a group, organization, or multi-party entity. It has:
- Profile (kind:0) - name, description, picture
- Metadata (kind:39000) - stewards, policies
- Commons - content published in its name

### Commons

The collection of events belonging to a collective. Identified by the `["commons", "<collective_npub>"]` tag.

### Cap (Capability)

A signed grant authorizing specific actions, issued by the collective, a steward, or a delegator:
- `publish` - Create events of specified kinds
- `delete` - Remove events (kind:5)
- `access` - Read private content (future)
- `delegate` - Issue attenuated caps to others

### Steward

A member with authority to govern the collective:
- Issues and revokes caps
- Manages collective profile
- Holds or shares access to collective's nsec

## Key Properties

### Backwards Compatible

Collectives are just npubs. Existing clients:
- Can follow them (no changes)
- Can display their posts (no changes)
- Can show their profile (no changes)

### Portable

Collective identity is not tied to any relay or platform. The npub works everywhere Nostr works.

### Composable

Collectives can connect by exchanging caps:
- Alliance: Mutual full access
- Federation: Shared membership
- Coalition: Time-limited collaboration

### Self-Sovereign

Collectives control their own:
- Membership policies (via caps)
- Content policies (via commons rules)
- Inter-group relationships (via cap exchange)

## Terminology

| Term | Definition |
|------|------------|
| **Collective** | An npub representing a group |
| **Commons** | Events belonging to a collective |
| **Cap** | A signed grant of actions |
| **Steward** | Member who governs the collective |
| **NosCAP** | The capability system for Nostr |

## Next Steps

- [NIP-A: Collective Identity](NIP-A-Collective-Identity.md) - Technical specification
- [NIP-B: NosCAP](NIP-B-NosCAP.md) - Capability system details
- [NIP-C: Commons Enforcement](NIP-C-Commons-Enforcement.md) - Relay validation
