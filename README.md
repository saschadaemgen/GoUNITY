<p align="center">
  <strong>GoUNITY</strong>
</p>

<p align="center">
  <strong>Verified identities for encrypted messaging. Privacy-preserving moderation for SimpleX.</strong><br>
  Optional verification. Effective moderation. Zero metadata leakage.
</p>

<p align="center">
  <a href="LICENSE"><img src="https://img.shields.io/badge/License-AGPL--3.0-blue.svg" alt="License"></a>
  <a href="#status"><img src="https://img.shields.io/badge/status-concept-yellow.svg" alt="Status"></a>
  <a href="https://github.com/saschadaemgen/SimpleGo"><img src="https://img.shields.io/badge/ecosystem-SimpleGo-green.svg" alt="SimpleGo"></a>
</p>

---

GoUNITY is a verified identity and moderation system for the SimpleX messaging ecosystem. It allows users to optionally register a verified username and present it across SimpleX groups, shops, and contacts - without revealing their SimpleX addresses, queues, or message history to anyone.

GoUNITY solves the biggest unsolved problem in encrypted messaging: **how do you moderate groups and prevent abuse when everyone is anonymous?**

The answer: give users the *choice* between anonymity and verified identity. Groups can then decide their own rules - from fully anonymous to fully verified, with everything in between.

GoUNITY works with the **standard, unmodified SimpleX app** through [GoBot](https://github.com/saschadaemgen/GoBot) - a moderation bot that verifies certificates and enforces bans directly in SimpleX groups. No plugins, no app modifications, no forks. Users keep their existing SimpleX app and interact with GoBot through normal chat messages.

---

## The problem

SimpleX Chat provides excellent privacy: no user IDs, no phone numbers, ephemeral profiles. But this creates a fundamental moderation problem:

```
User gets banned from group
  -> Creates new SimpleX profile (takes 3 seconds)
  -> Rejoins group as "someone else"
  -> Ban was pointless
```

Every SimpleX group admin knows this pain. Spam, harassment, ban evasion - there is currently no effective solution. SimpleX's design philosophy (no persistent identity) directly conflicts with the need for accountability in communities.

Telegram solved this with phone numbers. Discord solved this with accounts. Both solutions destroy privacy. GoUNITY solves it while preserving privacy.

---

## How GoUNITY works

### The verified username

A GoUNITY username is a cryptographically signed certificate. Think of it as a digital passport for SimpleX - it proves "this person registered this name" without revealing who they are or what they do on SimpleX.

```
                GoUNITY                              SimpleX
            Identity Service                        Network
          
  User registers                          User chats, joins groups,
  "MeinPrinz"                             shops at GoShop
  
  Pays 5 EUR/year                         Uses fresh queues per contact
  Verifies email                          All messages E2E encrypted
  
  Receives signed                         Presents certificate when
  certificate (200 bytes)                 joining verified groups
          |                                         |
          |         NO CONNECTION BETWEEN            |
          |         THESE TWO SYSTEMS                |
          v                                         v
  GoUNITY DB knows:                       Group knows:
  - "MeinPrinz" = verified                - "MeinPrinz" = member
  - email: m***@***.de                    - certificate is valid
  - paid until: 2027-03                   - can be banned by name
  
  GoUNITY DB does NOT know:              Group does NOT know:
  - SimpleX addresses                    - real email or identity
  - which groups user is in              - payment information
  - what user writes                     - other group memberships
  - who user talks to                    - SimpleX queue addresses
```

### The certificate

A GoUNITY certificate is a small signed data structure (~300 bytes) that lives inside the user's SimpleX profile or is submitted as a chat message to [GoBot](https://github.com/saschadaemgen/GoBot):

```
GoUNITY Verified Username Certificate v1:
{
  "v": 1,
  "username": "MeinPrinz",
  "level": "verified",
  "features": ["groups", "shops", "contacts"],
  "issued": "2026-03-31T10:00:00Z",
  "expires": "2027-03-31T10:00:00Z",
  "issuer": "id.simplego.dev",
  "pubkey": "<user's Ed25519 public key>",
  "signature": "<Ed25519 signature by GoUNITY>"
}
```

**Why it cannot be faked:** The signature can only be created by GoUNITY's private key. Any SimpleX client can verify it using GoUNITY's public key (published openly). Forging a certificate requires breaking Ed25519 - computationally infeasible.

**Why it cannot be traced:** The certificate contains zero information about SimpleX queues, contacts, groups, or message history. Presenting it to a group reveals only: "I own the name MeinPrinz and it's currently valid."

### Verification levels

| Level | Verification | Price | Use case |
|:------|:-------------|:------|:---------|
| Basic | Email only | Free / 2 EUR/year | Casual users, communities |
| Verified | Email + phone | 5 EUR/year | Group moderators, regular users |
| Business | Email + phone + company | 15 EUR/year | Shops, services, organizations |
| Premium | Full KYC (ID document) | 30 EUR/year | Financial services, healthcare |

Higher levels unlock additional trust badges and features. A "Business" user in a GoShop listing carries more trust than a "Basic" user. Groups can set minimum verification levels.

---

## How verification works in practice

GoUNITY verification reaches SimpleX users through three channels:

### Channel 1: GoBot (standard SimpleX app - no modifications)

[GoBot](https://github.com/saschadaemgen/GoBot) is a moderation bot that runs as a headless SimpleX client. It joins groups as an admin member and handles all verification and moderation through normal chat messages. This is the primary channel because it works with every SimpleX app on every platform today.

```
User joins group (standard SimpleX app)
  |
  v
GoBot: "Welcome! This group requires GoUNITY verification.
        Please send your certificate with /verify <certificate>
        Get one at https://id.simplego.dev/register"
  |
  v
User sends: /verify eyJ2IjoxLCJ1c2VybmFtZSI6Ik1l...
  |
  v
GoBot verifies:
  1. Ed25519 signature valid?          -> YES
  2. Certificate expired?              -> NO  
  3. Username on ban list?             -> NO
  4. Minimum verification level met?   -> YES
  |
  v
GoBot: "Verified! Welcome, MeinPrinz."
GoBot grants full messaging permissions.
```

**No app modification needed.** Users keep their standard SimpleX app. GoBot handles everything.

### Channel 2: GoChat (browser widget - native support)

[GoChat](https://github.com/saschadaemgen/GoChat) has native GoUNITY support. The certificate is stored in the connection profile and verified locally in the browser using the GoUNITY public key. Verification badges appear next to usernames automatically.

```
[ MeinPrinz (Verified) ]: Hello, I'd like to order the ESP32 board.
[ Anonymous User ]: Is this available in blue?
[ TechShop (Business) ]: Yes! Both colors in stock.
```

### Channel 3: SimpleGo hardware (ESP32-S3 - native support)

[SimpleGo](https://github.com/saschadaemgen/SimpleGo) terminals verify certificates locally using hardware Ed25519 implementation. A verification badge appears on the display next to verified contacts.

---

## Group moderation

GoUNITY transforms SimpleX groups from unmoderable anonymous spaces into communities with real accountability - without sacrificing the privacy of honest participants. All moderation is executed by [GoBot](https://github.com/saschadaemgen/GoBot).

### Group modes

```
Group settings (via GoBot /mode command):

  ( ) Open             Anyone can join, no verification needed
  ( ) Mixed            Unverified users allowed with restrictions
  (x) Verified only    Only GoUNITY-verified users can join
  ( ) Invite only      Verified users, by invitation
```

### Restrictions for unverified users (Mixed mode)

| Restriction | Options |
|:------------|:--------|
| Messages per hour | 1 / 5 / 10 / unlimited |
| Send files | Yes / No |
| Send links | Yes / No |
| Send images | Yes / No |
| Voice messages | Yes / No |
| Reply to threads | Yes / No |

### Moderation actions

Admins issue moderation commands through GoBot:

```
/ban MeinPrinz harassment         Ban user permanently
/mute MeinPrinz 24h               Mute for 24 hours (read-only)
/restrict MeinPrinz 5/h           Limit to 5 messages per hour
/warn MeinPrinz                   Issue tracked warning
/unban MeinPrinz                  Remove ban
/unmute MeinPrinz                 Remove mute
/status MeinPrinz                 Show moderation history
/banlist                          Show all active bans
/reports                          Show pending user reports
/mode verified                    Set group to verified-only
```

Users can also interact:

```
/verify <certificate>             Submit GoUNITY certificate
/report MeinPrinz spam            Report a user to admins
/mystatus                         Check own verification status
/rules                            Show group rules
```

### Why bans actually work

```
WITHOUT GoUNITY:
  "ToxicUser" banned -> new profile "NiceGuy123" -> rejoins -> repeats
  
WITH GoUNITY (verified-only group via GoBot):
  "ToxicUser" banned -> 
    Same certificate? -> GoBot checks ban list -> REJECTED
    New certificate? -> Costs money + new verification + new username
    Anonymous profile? -> GoBot: group is verified-only -> REJECTED
    
  Ban evasion cost: 5-30 EUR + new email/phone + waiting period
  Ban evasion cost without GoUNITY: 3 seconds (new profile)
```

### Auto-moderation (GoBot)

GoBot can act autonomously based on configurable rules:

```
Auto-moderation rules:

  Spam detection:
    [x] Auto-mute after 10 messages in 60 seconds
    [x] Auto-delete messages with known spam patterns
    [ ] Auto-ban after 3 auto-mutes
    
  New member restrictions:
    [x] Read-only for first 5 minutes
    [x] No files for first 24 hours
    [x] No links for first 24 hours
    
  Flood protection:
    [x] Max 30 messages per hour per user
    [x] Max 5 images per hour per user
    [ ] Slow mode: 1 message per 30 seconds
```

---

## Privacy architecture

### The separation principle

GoUNITY's privacy model rests on one fundamental principle: **no single system holds enough information to identify a user's complete activity.**

```
+------------------+     +------------------+     +------------------+
|   GoUNITY SIS    |     |   SimpleX/SMP    |     |     GoShop       |
|                  |     |                  |     |                  |
| username         |     | anonymous queues |     | username         |
| email            |     | E2E messages     |     | order history    |
| payment          |     | group membership |     | delivery address |
| verification     |     | contact list     |     | payment status   |
|                  |     |                  |     |                  |
| Does NOT know:   |     | Does NOT know:   |     | Does NOT know:   |
| - SMP queues     |     | - real identity  |     | - SMP queues     |
| - groups         |     | - email/phone    |     | - email/phone    |
| - messages       |     | - payment info   |     | - other groups   |
| - orders         |     | - orders         |     | - real identity  |
+------------------+     +------------------+     +------------------+
        |                        |                        |
        |    CRYPTOGRAPHIC SEPARATION - NO LINKAGE        |
        |    Even with access to all three databases,     |
        |    correlation requires breaking E2E encryption  |
```

### What an attacker learns from each system

| Attacker compromises... | They learn... | They do NOT learn... |
|:------------------------|:-------------|:---------------------|
| GoUNITY database | Usernames, emails, payment | Messages, contacts, groups, orders |
| SMP relay server | Encrypted blobs, queue IDs | Who owns which queue, message content |
| GoShop database | Usernames, orders, addresses | Email, phone, SMP queues, messages |
| GoBot instance | Group members, bans, usernames | Email, phone, payment, other groups |
| GoUNITY + SMP server | Still nothing new | Certificate has no SMP queue info |
| GoUNITY + GoShop | Username links to orders | Still no SMP queues or messages |
| All systems | Username + orders + email | Message content (E2E encrypted) |

Even compromising all systems simultaneously does not reveal message content - that requires breaking the Double Ratchet encryption on the user's device.

### Zero-knowledge certificate verification

When a user presents their GoUNITY certificate (via GoBot or GoChat), verification happens locally:

```
1. User sends certificate to GoBot (or GoChat verifies in-browser)
2. Verifier independently:
   a. Checks: Is the signature valid? (Ed25519 verify with GoUNITY public key)
   b. Checks: Is the certificate expired?
   c. Checks: Is the username on the ban list?
3. No network request to GoUNITY needed
4. GoUNITY never learns which groups the user joins
```

This is the key privacy property: **GoUNITY issues certificates but never learns where they are used.** The verification is offline-capable and does not create a usage trail.

### Certificate revocation

If a user's account is suspended (fraud, payment failure), GoUNITY publishes a Certificate Revocation List (CRL) - a signed list of revoked usernames. GoBot and GoChat periodically fetch this list (e.g. daily) to ensure revoked users are detected even before their certificate technically expires.

The CRL reveals only which usernames are revoked - not why, not where they were used, not who reported them.

---

## GoShop integration

GoUNITY verified usernames create trust in encrypted e-commerce:

```
Customer "MeinPrinz" (Verified) browses GoShop:
  -> Sees product, clicks "Buy"
  -> GoChat opens encrypted channel to shop owner
  -> Sends: order + delivery address (E2E encrypted)
  -> Shop owner sees: "MeinPrinz (Verified)" ordered Product X
  
Shop owner "TechShop" (Business Verified):
  -> Customer sees Business badge = legitimate shop
  -> Encrypted order confirmation sent back
  -> Delivery updates via encrypted channel
  
Neither GoUNITY nor the SMP server sees the order.
The shop's hosting provider sees only encrypted blocks.
```

For GoShop groups moderated by GoBot, the bot can also assist with order verification - confirming that customers meet the shop's minimum verification level before processing orders.

**Trust badges in GoShop:**

| Badge | Meaning | Benefit |
|:------|:--------|:-------|
| No badge | Anonymous customer | Privacy, but shop may require verification |
| Verified | Email + phone confirmed | Shops can accept orders with confidence |
| Business | Company verified | Customers trust the shop is legitimate |

---

## Technical architecture

### Components

```
GoUNITY Ecosystem:

+------------------------+
|   GoUNITY SIS          |     SIS = SimpleGo Identity Service
|   (id.simplego.dev)    |
|                        |
|   REST API + Database  |
|   Certificate issuing  |
|   Username registry    |
|   Payment processing   |
|   Revocation lists     |
+----------+-------------+
           |
           | HTTPS (registration, renewal, CRL)
           |
+----------+---+------------+-------------------+
|              |             |                    |
v              v             v                    v
+---------+  +---------+  +---------+  +------------------+
| GoChat  |  | GoBot   |  | SimpleGo|  | SimpleX App      |
| browser |  | server  |  | ESP32   |  | (via GoBot)      |
|         |  |         |  |         |  |                  |
| Native  |  | Verify  |  | Native  |  | No modification  |
| verify  |  | via chat|  | verify  |  | needed - GoBot   |
| + badge |  | + mod   |  | + badge |  | handles it all   |
+---------+  +---------+  +---------+  +------------------+
```

**GoBot is the critical bridge.** It enables GoUNITY verification for the millions of existing SimpleX users without requiring any app changes. Users interact with GoBot through standard chat messages. GoBot verifies certificates locally and enforces moderation actions in real-time.

### Technology stack

| Component | Technology | Reason |
|:----------|:-----------|:-------|
| SIS Backend | Go | Matches GoRelay, strong crypto stdlib, single binary |
| Database | PostgreSQL | Mature, ACID, good for usernames + certificates |
| API | REST + JSON | Simple, universal, easy to integrate |
| Certificate signing | Ed25519 | Fast, small signatures (64 bytes), audited |
| Payment | Stripe / crypto | Both options for accessibility |
| Hosting | Self-hosted (Hetzner) | EU data sovereignty, GDPR compliant |
| CRL distribution | Signed JSON over HTTPS | Simple, cacheable, offline-verifiable |
| GoBot | Go + simplex-chat CLI | Headless SimpleX client for group moderation |

### API endpoints (draft)

```
POST   /v1/register          Register username + start verification
POST   /v1/verify/email      Confirm email verification code
POST   /v1/verify/phone      Confirm phone verification code
POST   /v1/certificate       Issue signed certificate (after verification + payment)
POST   /v1/renew             Renew certificate before expiration
POST   /v1/revoke            Revoke own certificate (voluntary)
GET    /v1/check/{username}  Check if username is taken
GET    /v1/crl               Get current Certificate Revocation List
GET    /v1/pubkey            Get GoUNITY's public verification key
```

---

## Roadmap

| Phase | Focus | Status |
|:------|:------|:-------|
| 0 | Concept + Architecture | DONE |
| 1 | SIS Backend (Go): registration, verification, certificate issuing | Planned |
| 2 | GoBot integration: certificate verification in SimpleX groups | Planned |
| 3 | GoChat integration: certificate in profile, badge display | Planned |
| 4 | Group moderation via GoBot: verified-only mode, ban/mute/restrict | Planned |
| 5 | GoShop integration: verified checkout, trust badges | Planned |
| 6 | SimpleGo hardware: badge display on ESP32-S3 | Planned |
| 7 | Advanced: cross-group bans, reputation, dispute resolution | Future |

---

## SimpleGo ecosystem

GoUNITY is the identity layer of the SimpleGo ecosystem for encrypted communication.

| Project | What it does | Repository |
|:--------|:-------------|:-----------|
| **[SimpleGo](https://github.com/saschadaemgen/SimpleGo)** | Dedicated hardware messenger on ESP32-S3 | [SimpleGo](https://github.com/saschadaemgen/SimpleGo) |
| **[GoRelay](https://github.com/saschadaemgen/GoRelay)** | Encrypted relay server (SMP + GRP) | [GoRelay](https://github.com/saschadaemgen/GoRelay) |
| **[GoChat](https://github.com/saschadaemgen/GoChat)** | Browser-native encrypted chat plugin | [GoChat](https://github.com/saschadaemgen/GoChat) |
| **[GoShop](https://github.com/saschadaemgen/GoShop)** | End-to-end encrypted e-commerce | [GoShop](https://github.com/saschadaemgen/GoShop) |
| **[GoBot](https://github.com/saschadaemgen/GoBot)** | Moderation + automation bot for SimpleX | [GoBot](https://github.com/saschadaemgen/GoBot) |
| **[GoUNITY](https://github.com/saschadaemgen/GoUNITY)** | Verified identity + moderation (this project) | [GoUNITY](https://github.com/saschadaemgen/GoUNITY) |

**GoUNITY issues the passports. GoBot checks them at the door.**

---

## Why "GoUNITY"

**Unity** - bringing together identity and anonymity, moderation and privacy, community and individual freedom. The name reflects the core mission: unifying the SimpleX ecosystem under an optional identity layer that respects everyone's choice.

**Go** - part of the SimpleGo family. Written in Go. Ready to go.

---

## Status

Concept phase. Architecture document complete. Implementation begins with Phase 1 (SIS Backend).

---

## License

AGPL-3.0

---

<p align="center">
  <i>GoUNITY is part of the <a href="https://github.com/saschadaemgen/SimpleGo">SimpleGo ecosystem</a> by IT and More Systems, Recklinghausen, Germany.</i>
</p>

<p align="center">
  <strong>GoUNITY - verified identity for encrypted messaging. Your name, your choice, your privacy.</strong>
</p>
