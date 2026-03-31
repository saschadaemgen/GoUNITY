# GoUNITY - Technical Concept
# Verified Identity and Moderation for the SimpleX Ecosystem

**Project:** GoUNITY - SimpleGo Identity Service (SIS)
**Author:** Sascha Daemgen / IT and More Systems
**Date:** 2026-03-31
**Status:** Concept Phase

---

## 1. Problem statement

SimpleX Chat provides strong privacy through ephemeral profiles and
anonymous queues. This design is excellent for individual privacy but
creates an unsolvable moderation problem for group communication:

1. **Ban evasion is trivial.** Creating a new SimpleX profile takes
   seconds. Any banned user can rejoin immediately with a fresh identity.

2. **Spam is unstoppable.** Without persistent identity, rate limiting
   per-user is meaningless. A spammer creates N profiles and sends N
   times the limit.

3. **Trust is impossible.** In a marketplace (GoShop), neither buyer
   nor seller can verify the other's legitimacy. Scam accounts are
   indistinguishable from genuine ones.

4. **Accountability is absent.** Harassment, doxxing, and abuse have
   no consequences when the abuser can discard their identity instantly.

No existing solution addresses this within the SimpleX ecosystem.
Telegram's solution (phone numbers) and Discord's solution (accounts)
both destroy privacy. GoUNITY solves the problem while preserving
the privacy guarantees that make SimpleX valuable.

---

## 2. Core concept: optional verified identity

GoUNITY introduces an **optional** identity layer. Users choose:

- **Anonymous** (default): Standard SimpleX behavior. No registration,
  no verification, no persistent identity. Full privacy.

- **Verified**: Registered username with cryptographic certificate.
  Persistent identity across sessions. Can be moderated, banned, and
  trusted. Privacy preserved through cryptographic separation.

Both modes coexist. A user can have anonymous profiles AND a verified
profile simultaneously. Groups decide which modes they accept.

---

## 3. Certificate design

### 3.1 Certificate structure

A GoUNITY certificate is a compact signed data structure:

```
GoUNITY Certificate v1 (binary encoding):

[1B version=1]
[1B level]                    0x01=Basic, 0x02=Verified, 0x03=Business, 0x04=Premium
[1B usernameLen][username]    UTF-8, max 32 characters
[8B issuedAt]                 Unix timestamp (seconds)
[8B expiresAt]                Unix timestamp (seconds)
[32B userPubKey]              User's Ed25519 public key
[32B issuerPubKey]            GoUNITY SIS public key
[64B signature]               Ed25519 signature over all preceding fields
```

**Total size:** ~150-180 bytes (depending on username length)

### 3.2 JSON representation (for transport via GoBot or GoChat)

```json
{
  "gounity": {
    "v": 1,
    "username": "MeinPrinz",
    "level": "verified",
    "issued": "2026-03-31T10:00:00Z",
    "expires": "2027-03-31T10:00:00Z",
    "pubkey": "base64url(32 bytes)",
    "issuer": "base64url(32 bytes)",
    "sig": "base64url(64 bytes)"
  }
}
```

This JSON block can be stored in the SimpleX profile's custom fields
(for GoChat) or submitted as a chat message to GoBot (for the
standard SimpleX app).

### 3.3 Certificate verification (local, offline)

```
verify(certificate):
  1. Decode all fields
  2. Check: expiresAt > now()           (not expired)
  3. Check: issuerPubKey == known SIS key (correct issuer)
  4. Compute: dataToVerify = all fields except signature
  5. Check: Ed25519.verify(signature, dataToVerify, issuerPubKey)
  6. Return: valid / invalid
```

No network request needed. The GoUNITY public key is embedded in
every verifier (GoBot, GoChat, SimpleGo hardware). Verification
works offline, in airplane mode, behind firewalls.

### 3.4 Certificate properties

| Property | Value |
|:---------|:------|
| Signature algorithm | Ed25519 (RFC 8032) |
| Signature size | 64 bytes |
| Key size | 32 bytes (public), 64 bytes (private) |
| Forgery resistance | 2^128 (computational security) |
| Offline verification | Yes (no network needed) |
| Expiration | Configurable (default: 1 year) |
| Revocation | Via Certificate Revocation List (CRL) |

---

## 4. Privacy model

### 4.1 The separation principle

GoUNITY's privacy guarantee:

**Registration data (GoUNITY knows):**
- Username
- Email address
- Phone number (if Verified+)
- Payment information
- Verification timestamp

**Usage data (GoUNITY does NOT know):**
- Which SimpleX profiles use this certificate
- Which groups the user joins
- What messages the user sends
- Who the user talks to
- What the user buys in GoShop
- Which devices the user has

This separation is enforced cryptographically: the certificate
contains no SimpleX-specific identifiers. There is no field for
queue IDs, server addresses, or connection tokens.

### 4.2 Threat model

| Threat | Mitigation |
|:-------|:-----------|
| GoUNITY server compromised | Attacker gets usernames + emails. Cannot link to SimpleX activity. |
| SMP server compromised | Attacker gets encrypted blobs + queue IDs. Cannot link to GoUNITY usernames. |
| GoShop database compromised | Attacker gets usernames + orders. Cannot get email/phone (stored at GoUNITY). |
| GoBot instance compromised | Attacker gets group members + bans. Cannot get email/phone/payment or other groups. |
| Man-in-the-middle | Certificate signature prevents forgery. E2E encryption prevents content interception. |
| Rogue GoUNITY operator | Can issue fake certificates. Mitigated by certificate transparency log (Phase 7). |
| State-level adversary | Can potentially correlate timing across systems. Mitigated by cover traffic (GRP profile). |

### 4.3 What GoUNITY intentionally cannot do

Even if GoUNITY wanted to (or was forced by a court):

- **Cannot read messages.** Messages are E2E encrypted between users.
  GoUNITY has no keys.
- **Cannot identify group members.** GoUNITY doesn't know which
  groups a user joins. GoBot stores member lists locally on the
  admin's server, not on GoUNITY.
- **Cannot link purchases.** GoShop orders are E2E encrypted to the
  shop owner. GoUNITY has no access.
- **Cannot track devices.** No device fingerprinting, no IP logging
  beyond registration.

---

## 5. Enforcement: GoBot

### 5.1 The bridge to standard SimpleX

GoUNITY issues certificates, but certificates alone are useless
without enforcement. [GoBot](https://github.com/saschadaemgen/GoBot)
is the enforcement arm:

```
GoUNITY issues passports.
GoBot checks them at the door.
```

GoBot runs as a headless SimpleX client (via simplex-chat CLI).
It joins groups as an admin member and handles all verification and
moderation through normal chat messages. Users interact with GoBot
using their standard, unmodified SimpleX app.

### 5.2 How GoBot verifies certificates

```
User joins group -> GoBot greets and asks for certificate
User sends: /verify eyJ2IjoxLC...
GoBot:
  1. Decodes certificate from base64
  2. Verifies Ed25519 signature (GoUNITY public key)
  3. Checks expiration date
  4. Checks local CRL (Certificate Revocation List)
  5. Checks group ban list
  6. Checks minimum verification level
  -> All pass: "Welcome, MeinPrinz!"
  -> Any fail: explains reason, restricts or removes user
```

**Critical privacy property:** GoBot verifies locally. No request
to GoUNITY during verification. GoUNITY never learns where a
certificate is used.

### 5.3 GoBot moderation capabilities

| Action | Command | Effect |
|:-------|:--------|:-------|
| Ban | /ban username reason | Permanent removal, cannot rejoin |
| Mute | /mute username 24h | Read-only for duration |
| Restrict | /restrict username 5/h | Message rate limit |
| Warn | /warn username | Tracked warning (visible to admins) |
| Report | /report username reason | User-initiated report to admins |
| Mode | /mode verified | Set group to verified-only |

### 5.4 CRL synchronization

GoBot periodically fetches the Certificate Revocation List:

```
Daily (configurable):
  1. HTTPS GET https://id.simplego.dev/v1/crl
  2. Verify CRL signature (GoUNITY public key)
  3. Cross-reference with active group members
  4. Notify/remove members with revoked certificates
```

This is the only regular contact between GoBot and GoUNITY.
It reveals nothing about which groups GoBot moderates.

---

## 6. Moderation system

### 6.1 Group moderation model

Groups maintain a local moderation database (stored on the GoBot
server, synced to co-admins via SMP):

```
Group Moderation State:
{
  "mode": "verified_only",
  "restrictions_unverified": {
    "messages_per_hour": 5,
    "send_files": false,
    "send_links": false
  },
  "bans": [
    {"username": "ToxicUser", "reason": "harassment", "date": "2026-03-31", "admin": "GroupOwner"},
    {"username": "SpamBot99", "reason": "spam", "date": "2026-03-30", "admin": "Moderator1"}
  ],
  "mutes": [
    {"username": "LoudPerson", "until": "2026-04-01T12:00:00Z", "admin": "Moderator2"}
  ],
  "warnings": [
    {"username": "EdgeCase", "count": 2, "last": "2026-03-29"}
  ]
}
```

### 6.2 Ban enforcement flow

```
"ToxicUser" tries to join a verified-only group:

1. "ToxicUser" presents GoUNITY certificate via /verify
2. GoBot checks:
   a. Certificate valid? -> YES (signature OK, not expired)
   b. Username on ban list? -> YES ("ToxicUser" banned)
   c. Result: REJECT entry
3. "ToxicUser" cannot join

"ToxicUser" creates new SimpleX profile "NiceGuy":
4. No GoUNITY certificate -> Group is verified-only -> GoBot REJECTS
5. "ToxicUser" cannot join without a valid certificate

"ToxicUser" registers new GoUNITY username "NiceGuy2":
6. Costs money (5-30 EUR)
7. Requires new email/phone verification
8. Waiting period (optional, configurable by admin)
9. Previous ban history visible to admins (if cross-reference enabled)
```

### 6.3 Report system

```
User A reports User B in a group:

1. User A sends: /report MeinPrinz harassment
2. GoBot forwards report to group admins (via DM)
3. Admins see: reporter, reported user, reason, recent messages
4. Admin takes action: /warn, /mute, /restrict, /ban, or dismisses
5. Action executed by GoBot in real-time
```

Reports stay within the group. GoUNITY SIS never sees reports.
All moderation is decentralized - the group admins decide.

### 6.4 Cross-group ban sharing (optional, Phase 7)

Groups can optionally publish their ban lists (signed by GoBot
using the group admin's key). Other GoBot instances can subscribe:

```
"Anti-Spam Coalition" ban list:
  Signed by: GroupAdmin1, GroupAdmin2, GroupAdmin3
  Bans: ["SpamBot1", "SpamBot2", "Scammer99", ...]

GoBot config:
  shared_ban_lists:
    - url: "https://bans.simplego.dev/anti-spam.json"
      trust: true
```

This creates a decentralized, opt-in reputation system without any
central authority controlling who gets banned.

---

## 7. GoShop integration

### 7.1 Verified commerce

```
GoShop listing:
  Product: "ESP32-S3 Development Board"
  Price: 24.99 EUR
  Seller: "TechShop" (Business Verified)
  
  [Buy Now] button triggers:
  1. GoChat opens encrypted channel to "TechShop"
  2. Customer sends order details (E2E encrypted)
  3. Order includes GoUNITY username (if verified)
  4. Shop can require verified customers for high-value orders
  5. GoBot in shop group confirms customer verification level
```

### 7.2 Trust tiers for shops

| Customer level | Shop may allow... |
|:--------------|:------------------|
| Anonymous | Browse products, ask questions |
| Basic | Small orders (< 50 EUR) |
| Verified | Standard orders |
| Premium | High-value orders, credit/invoice |

Each shop decides its own policy. GoUNITY provides the verification
infrastructure; GoBot enforces it; shops decide how to use it.

---

## 8. SIS Backend architecture

### 8.1 Technology choices

| Component | Choice | Reasoning |
|:----------|:-------|:----------|
| Language | Go | Matches GoRelay + GoBot, excellent crypto stdlib, single binary |
| Database | PostgreSQL | ACID transactions for username uniqueness, mature, reliable |
| API | REST + JSON | Universal, easy to integrate from any client |
| Signing | Ed25519 (Go crypto/ed25519) | FIPS-quality implementation in Go stdlib |
| Payment | Stripe | Established, reliable, EU-compliant |
| Email verification | SMTP (Postmark/SES) | Reliable delivery |
| Phone verification | Twilio | Industry standard |
| Hosting | Hetzner (Germany) | EU data sovereignty, GDPR, affordable |
| TLS | Let's Encrypt | Free, automated |

### 8.2 Database schema (draft)

```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username        VARCHAR(32) UNIQUE NOT NULL,
    email_hash      BYTEA NOT NULL,          -- SHA-256(email), not plaintext!
    phone_hash      BYTEA,                   -- SHA-256(phone), optional
    level           SMALLINT NOT NULL,        -- 1=Basic, 2=Verified, 3=Business, 4=Premium
    ed25519_pubkey  BYTEA NOT NULL,          -- 32 bytes
    created_at      TIMESTAMPTZ NOT NULL,
    verified_at     TIMESTAMPTZ,
    paid_until      TIMESTAMPTZ,
    status          SMALLINT NOT NULL,        -- 1=pending, 2=active, 3=suspended, 4=revoked
    revocation_reason VARCHAR(256)
);

CREATE TABLE certificates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID REFERENCES users(id),
    certificate     BYTEA NOT NULL,          -- The signed certificate blob
    issued_at       TIMESTAMPTZ NOT NULL,
    expires_at      TIMESTAMPTZ NOT NULL,
    revoked_at      TIMESTAMPTZ,
    version         SMALLINT NOT NULL
);

CREATE TABLE audit_log (
    id              BIGSERIAL PRIMARY KEY,
    timestamp       TIMESTAMPTZ NOT NULL,
    action          VARCHAR(32) NOT NULL,     -- register, verify, issue, revoke, renew
    user_id         UUID,
    details         JSONB
);
```

**Privacy note:** Email and phone are stored as SHA-256 hashes, not
plaintext. This prevents mass data exfiltration while still allowing
duplicate detection (same email cannot register twice).

### 8.3 API specification (draft)

```
POST   /v1/register          Register username + start verification
POST   /v1/verify/email      Confirm email verification code
POST   /v1/verify/phone      Confirm phone verification code
POST   /v1/pay               Process payment via Stripe
POST   /v1/certificate       Issue signed certificate (after verification + payment)
POST   /v1/renew             Renew certificate before expiration
POST   /v1/revoke            Revoke own certificate (voluntary)
GET    /v1/check/{username}  Check if username is taken
GET    /v1/crl               Get current Certificate Revocation List
GET    /v1/pubkey            Get GoUNITY's public verification key
```

---

## 9. Client integration matrix

### 9.1 Three verification channels

| Client | How verification works | App modification needed? |
|:-------|:----------------------|:------------------------|
| **GoChat** (browser) | Certificate in profile JSON, local Ed25519 verify, badge display | No (native support) |
| **GoBot** (any SimpleX app) | User sends /verify command with certificate text, GoBot verifies | No (standard app) |
| **SimpleGo** (ESP32-S3) | Certificate in connection data, hardware Ed25519 verify, badge on screen | No (native support) |

### 9.2 GoChat integration

GoChat stores the certificate in the connection profile and verifies
locally using the GoUNITY public key embedded in the JavaScript bundle:

```javascript
// GoChat certificate verification (browser)
import { ed25519 } from '@noble/curves/ed25519';

function verifyGoUNITY(cert, gounityPubKey) {
  const dataToVerify = cert.slice(0, -64);  // everything except signature
  const signature = cert.slice(-64);         // last 64 bytes
  return ed25519.verify(signature, dataToVerify, gounityPubKey);
}
```

### 9.3 GoBot integration

GoBot wraps the simplex-chat CLI and processes /verify commands:

```go
// GoBot certificate verification (server)
import "crypto/ed25519"

func verifyGoUNITY(cert []byte, gounityPubKey ed25519.PublicKey) bool {
    dataToVerify := cert[:len(cert)-64]
    signature := cert[len(cert)-64:]
    return ed25519.Verify(gounityPubKey, dataToVerify, signature)
}
```

### 9.4 SimpleGo hardware integration

The ESP32-S3 verifies certificates using mbedTLS Ed25519:

```c
// SimpleGo certificate verification (ESP32-S3)
int verify_gounity(const uint8_t *cert, size_t cert_len,
                   const uint8_t *gounity_pubkey) {
    size_t data_len = cert_len - 64;
    const uint8_t *sig = cert + data_len;
    return mbedtls_ed25519_verify(sig, gounity_pubkey, cert, data_len);
}
```

---

## 10. Revenue model

| Plan | Price | Features |
|:-----|:------|:---------|
| Basic | Free or 2 EUR/year | Username + email verification |
| Verified | 5 EUR/year | + phone verification + priority support |
| Business | 15 EUR/year | + company verification + GoShop trust badge |
| Premium | 30 EUR/year | + KYC verification + financial trust level |

**Projected break-even:** ~500 paying users at average 8 EUR/year
= 4,000 EUR/year. Server costs ~600 EUR/year (Hetzner). Sustainable
from day one with low user numbers.

---

## 11. Phased implementation plan

### Phase 0: Architecture (DONE)
- This concept document
- README.md with GoBot integration
- GoBot concept document

### Phase 1: SIS Backend (MVP)
- Go REST API with PostgreSQL
- Username registration + email verification
- Ed25519 certificate issuing
- Basic CRL endpoint
- Stripe payment integration
- Deployed on id.simplego.dev

### Phase 2: GoBot integration
- GoBot verifies certificates in SimpleX groups
- Ban enforcement by verified username
- Moderation commands (ban/mute/restrict/warn)
- CRL synchronization

### Phase 3: GoChat integration
- Certificate in SimpleX profile JSON
- Badge rendering in chat UI
- Local certificate verification
- "Verified" indicator next to username

### Phase 4: Group moderation (advanced)
- Auto-moderation (spam, flood, cooldown)
- Report system
- Multi-group GoBot management
- Admin web dashboard

### Phase 5: GoShop integration
- Verified checkout
- Trust badges on shop listings
- Customer verification requirements

### Phase 6: SimpleGo hardware
- Certificate verification on ESP32-S3
- Badge display on hardware screen
- Offline verification (embedded SIS pubkey)

### Phase 7: Advanced features
- Cross-group ban sharing
- Certificate transparency log
- Reputation scoring (opt-in)
- Dispute resolution protocol

---

## 12. Open questions

1. **SimpleX profile custom fields:** Does the current SimpleX profile
   format support arbitrary JSON fields? If not, GoBot's chat-based
   verification is the primary path (no profile modification needed).

2. **Certificate size budget:** SimpleX profiles are transmitted in
   16KB SMP blocks. A 300-byte certificate easily fits. But if we
   add features (reputation, badges, metadata), what's the safe limit?

3. **Multi-device support:** If a user has SimpleX on phone + desktop
   + GoChat in browser, each device submits the certificate
   independently to GoBot. GoBot stores the verification per group,
   not per device.

4. **Legal requirements:** For Business/Premium verification levels,
   what KYC requirements apply under EU law? Do we need a licensed
   identity provider?

5. **SimpleX community reaction:** Will Evgeny and the SimpleX community
   welcome or resist an optional identity layer? Positioning matters -
   GoBot approach is ideal because it requires zero upstream changes.

6. **simplex-chat CLI API:** Is the JSON API mode stable enough for
   GoBot production use? What happens on CLI version updates?

---

*GoUNITY Technical Concept v2 - March 2026*
*IT and More Systems, Recklinghausen, Germany*
