# Nostr Collectives: Proof of Concept Plan

## Goal

Build a client that demonstrates the core Collectives flow: identity management, cap issuance, and commons interaction with a caps-enabled relay.

---

## Reusable Components from SR-NOSTR-1

| Component | File | Reuse Level | Notes |
|-----------|------|-------------|-------|
| NostrManager | `nostr.js` | High | Key gen, signing, relay pool - extend for caps |
| nostr-tools | `nostr.bundle.js` | As-is | Library for Nostr primitives |
| PWA Shell | `index.html`, `manifest.json`, `sw.js` | High | Mobile-first, installable |
| Templates | `templates.js` | Medium | Good patterns, need new views |
| Config | `config.js` | High | Clean pattern for relay/key config |
| Relay Loader | `data-sources/relay-loader.js` | Medium | Adapt for commons queries |

---

## PoC Steps

### Step 1: Identity Wallet

Self-hosted key management for personal and collective identities.

- [ ] `IdentityManager` class
  - [ ] Generate/import personal nsec
  - [ ] Generate/import collective nsec(s)
  - [ ] Switch between identities
  - [ ] Store in localStorage (encrypted)
- [ ] Minimal UI: identity picker, key display

---

### Step 2: Issue + Send (Gift Wrap) CAP

Create and deliver capabilities to members.

- [ ] `CapEngine` class
  - [ ] `issueCap(collective, grantee, actions, expiry)` → kind:39100
  - [ ] Sign cap with collective's nsec
- [ ] Gift wrap cap via NIP-44 encrypted DM to grantee
- [ ] `CapStore` class
  - [ ] Store received caps locally
  - [ ] Retrieve cap for a given collective
- [ ] UI: issue cap form, received caps list

---

### Step 3: Configure Commons on Relay

Setup relay to enforce caps for a collective's commons.

- [ ] Relay configuration for enforced commons
- [ ] Validation logic:
  - [ ] Check `["commons", npub]` tag
  - [ ] Parse cap reference
  - [ ] Verify signature, grantee, expiry
- [ ] Return NOTICE on invalid cap

---

### Step 4: Post to Commons using CAP

Publish content to a collective's commons with cap attachment.

- [ ] `CommonsComposer` class
  - [ ] Create event with `["commons", "<collective_npub>"]` tag
  - [ ] Attach cap: `["cap", "<reference>"]`
  - [ ] Sign with member's nsec
  - [ ] Publish to relay
- [ ] UI: compose view with collective selector
- [ ] Display relay response (OK / rejection)

---

### Step 5: Reading from Commons using CAP

Subscribe to and display commons content (including private commons).

- [ ] Subscribe with `#commons` filter
- [ ] For private commons: AUTH with cap attachment
- [ ] Display posts grouped by collective
- [ ] Show author + cap validity indicator
- [ ] UI: commons feed view

---

## Event Kinds

| Kind | Name | Purpose |
|------|------|---------|
| 0 | Profile | Collective profile |
| 1 | Note | Content in commons |
| 39000 | Collective Metadata | Stewards, policies |
| 39100 | Cap Grant | Capability issuance |
| 39101 | Cap Revocation | Capability revocation |

---

## Tag Formats

```json
["commons", "<collective_npub>"]
["cap", "<cap_event_id>", "wss://relay.example.com"]
["cap", "noscap1<base64url_encoded_cap>"]
```
