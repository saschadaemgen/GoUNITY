# GoUNITY - Technical Concept
# Certificate Authority for SimpleX Identity and GoLab Communities

**Project:** GoUNITY (part of SimpleGo ecosystem)
**Author:** Sascha Daemgen / IT and More Systems
**Date:** April 16, 2026
**Status:** Season 1 - Concept

---

## 1. Overview

GoUNITY is a self-hosted Ed25519 certificate authority that
gives SimpleX Chat users and GoLab community members a
persistent, verifiable identity without creating accounts
on a server.

SimpleX Chat has the strongest privacy model of any messenger:
no user IDs, no metadata, pairwise queues, double ratchet
encryption. But this same privacy model means there is no way
to know who anyone is. Banned users rejoin in seconds. Verified
members cannot prove they are who they claim to be. Community
operators have no tools to build trust.

GoUNITY solves this with Ed25519 certificates. A certificate
binds a username to a cryptographic keypair. The certificate
is signed by a trusted CA (GoUNITY). Anyone can verify the
signature offline. Challenge-response proves the user holds
the private key - not just the certificate text. Bans are
linked to the certified username, not the temporary SimpleX
profile.

GoUNITY is a fork of smallstep/certificates (step-ca) - a
production-grade, open-source certificate authority in Go
with HSM support, Ed25519 native, and 8.1k GitHub stars.
We build the web frontend, payment integration, and
challenge-response logic on top.

---

## 2. The identity problem

### SimpleX groups today

SimpleX Chat groups have no identity layer. Any user can:
- Create a new profile with any name in seconds
- Join a group with the new profile
- Impersonate other members by copying their display name
- Rejoin after being kicked (new profile, new member ID)
- Create unlimited accounts at zero cost

This makes moderation impossible in any meaningful sense.
A ban on "Alice" means nothing when Alice can become "Bob"
in 10 seconds.

### GoLab communities need identity

GoLab combines GitLab-style project collaboration with
Twitter-style social feeds over SMP. Without persistent
identity, GoLab cannot support:
- Reputation (who has contributed over time)
- Accountability (who posted what)
- Access control (who can commit to a project)
- Trust (is this the same person I verified last week)

### What GoUNITY provides

A certificate that says: "This public key belongs to the
username MeinPrinz, verified by GoUNITY CA, valid until
April 2027." Anyone with the CA public key can verify this
statement offline. The user proves they hold the private
key through challenge-response. No server needs to be
contacted during verification.

---

## 3. Why step-ca

### Build vs. fork

Building a certificate authority from scratch would take
months and produce an inferior result. step-ca is battle-tested,
actively maintained, and provides everything GoUNITY needs:

| Need | step-ca solution |
|:-----|:----------------|
| Ed25519 certificates | Native support (`--kty OKP --curve Ed25519`) |
| HSM key protection | YubiKey PIV, PKCS#11 providers |
| Certificate revocation | CRL generation and HTTPS distribution |
| Custom certificate fields | X.509 OID extensions via templates |
| REST API | Built-in certificate CRUD |
| Database | BoltDB, Postgres, MySQL |
| Authentication | OIDC, JWK, X5C provisioners |
| Deployment | systemd, Docker, Kubernetes |

What step-ca does NOT provide (what we build):
- Web registration frontend (id.simplego.dev)
- Account management (email, password, payment)
- Challenge-response verification endpoint
- GoKey CRL sync mechanism
- DID:key identifier generation
- Device certificate workflow

### Fork strategy

GoUNITY is a fork, not a wrapper. We modify step-ca's
certificate templates and add custom provisioners. This
keeps us close to upstream for security updates while
allowing GoUNITY-specific extensions.

The Apache-2.0 license allows forking, modification, and
commercial use.

---

## 4. Design decisions

### 4.1 Ed25519 over RSA or ECDSA

Ed25519 is the only reasonable choice for GoUNITY:
- 32-byte public keys (vs 256+ bytes for RSA)
- 64-byte signatures (vs 256+ bytes for RSA)
- Deterministic signing (no RNG needed, no nonce reuse bugs)
- Constant-time by design (no timing side-channels)
- Fast on ESP32-S3 (26 ms for signing)
- Used by SimpleX, SSH, Tor, Signal, DNSSEC

RSA keys would not fit in ESP32 eFuse. ECDSA (P-256) has
a history of nonce-reuse vulnerabilities and requires
careful RNG implementation.

### 4.2 X.509 with custom OIDs

GoUNITY uses standard X.509 certificate format with custom
OID extensions for GoUNITY-specific fields. This was chosen
over custom certificate formats because:
- Standard tooling works (openssl, step CLI, browsers)
- CRL infrastructure is well-defined
- HSMs understand X.509 natively
- Interoperable with TLS, mTLS, HTTPS

Custom OIDs allow adding GoUNITY fields without conflicting
with standard X.509 extensions.

### 4.3 Registration fee as anti-spam

GoUNITY charges a fee for certificate registration. This is
not a revenue model - it is an anti-spam barrier.

Without a fee, a banned user creates a new email, gets a new
certificate, and re-enters the community. With a fee, ban
evasion has a financial cost that scales linearly with
repeated violations.

Pricing tiers are planned for different regions and use cases.
Free certificates may be available for verified open-source
contributors.

### 4.4 Offline verification

GoBot/GoKey verifies certificates without contacting GoUNITY.
This is critical for two reasons:
- GoUNITY downtime does not break verification
- GoUNITY never learns which groups or communities users join

The only online dependency is CRL sync (daily). Between syncs,
revocations have a propagation delay of up to 24 hours. This
is an accepted tradeoff - faster sync intervals are configurable.

### 4.5 DID:key for GoLab

GoLab uses W3C DID:key identifiers derived from GoUNITY
certificate public keys. This was chosen over custom identifier
formats because:
- Self-describing (the public key IS the identifier)
- W3C standard (interoperable with DID ecosystem)
- No resolution service needed (unlike did:web or did:ion)
- Deterministic (same key always produces same DID)

The human-readable username from the certificate is for display.
The DID:key is for cryptographic operations and message signing.

### 4.6 Device certificates for GoKey

GoUNITY issues device certificates that bind an ESP32's eFuse
Ed25519 key to a user identity. This was added to support
GoLab's hardware verification feature.

Device certificates are separate from identity certificates:
- Identity certificate: "MeinPrinz is verified"
- Device certificate: "This ESP32 belongs to MeinPrinz"

Revoking a device certificate does not revoke the identity.
A user can have multiple devices, each with its own certificate.

---

## 5. Certificate types

### 5.1 Identity certificate (primary)

Issued to every registered user. Used for SimpleX group
verification and GoLab community participation.

```json
{
  "subject": {"commonName": "MeinPrinz"},
  "extensions": [
    {"id": "1.3.6.1.4.1.XXXXX.1", "value": "MeinPrinz"},
    {"id": "1.3.6.1.4.1.XXXXX.2", "value": "verified"}
  ],
  "keyUsage": ["digitalSignature"],
  "extKeyUsage": ["clientAuth"],
  "notBefore": "2026-04-16T00:00:00Z",
  "notAfter": "2027-04-16T00:00:00Z"
}
```

### 5.2 Device certificate (optional)

Issued when a user binds a GoKey ESP32-S3 device.

```json
{
  "subject": {"commonName": "GoKey-001"},
  "extensions": [
    {"id": "1.3.6.1.4.1.XXXXX.1", "value": "MeinPrinz"},
    {"id": "1.3.6.1.4.1.XXXXX.3", "value": "device"},
    {"id": "1.3.6.1.4.1.XXXXX.4", "value": "sha256:efuse-pubkey-fingerprint"}
  ],
  "keyUsage": ["digitalSignature"],
  "notBefore": "2026-04-16T00:00:00Z",
  "notAfter": "2027-04-16T00:00:00Z"
}
```

### 5.3 Context certificate (future)

Planned for advanced GoLab use: per-community sub-certificates
signed by the user's identity key. Allows participation in
multiple communities without the communities being able to
link the same user across them (unlinkability).

---

## 6. What GoUNITY does NOT do

- **GoUNITY does not verify real-world identity.** It verifies
  email + payment. The username is a pseudonym, not a legal name.
- **GoUNITY does not track community activity.** It issues
  certificates. Where they are used is unknown to GoUNITY.
- **GoUNITY does not store private keys.** After delivery to the
  user, the private key is deleted from GoUNITY. Lost key = new
  certificate required.
- **GoUNITY does not perform moderation.** GoBot does. GoUNITY
  provides the identity that makes moderation persistent.
- **GoUNITY does not replace SimpleX privacy.** It adds optional
  persistent pseudonyms on top of SimpleX's anonymous transport.

---

## 7. Dependencies and timeline

| Dependency | Why GoUNITY needs it | Status |
|:-----------|:--------------------|:-------|
| step-ca (smallstep) | Base CA software | Available (Apache-2.0) |
| YubiKey | HSM for CA signing key | Available (hardware) |
| GoBot | Certificate verification in groups | In development (Season 2) |
| GoKey | Hardware verification + CRL sync | Planned (Season 3) |
| GoLab | Community identity consumer | Planned (concept) |

### Development phases

| Phase | Focus | Season |
|:------|:------|:-------|
| Phase 1 | Concept, architecture, documentation | Complete (this) |
| Phase 2 | step-ca fork, Ed25519 certificate template | Season 4 |
| Phase 3 | Web frontend (registration, account management) | Season 4 |
| Phase 4 | Challenge-response endpoint | Season 4 |
| Phase 5 | Payment integration | Season 4 |
| Phase 6 | GoKey CRL sync | Season 4 |
| Phase 7 | DID:key generation for GoLab | Season 5+ |
| Phase 8 | Device certificate workflow | Season 5+ |
| Phase 9 | Context certificates (unlinkability) | Future |

---

## 8. Comparable systems

| System | What it does | How GoUNITY differs |
|:-------|:------------|:-------------------|
| Let's Encrypt | Automated TLS certificates | GoUNITY: identity certificates, not TLS |
| Keybase | Signed identity chains | GoUNITY: no central server for verification |
| PGP/GPG | Web of trust | GoUNITY: CA-based, simpler trust model |
| Nostr NIP-05 | DNS-based identity verification | GoUNITY: certificate-based, offline-verifiable |
| Matrix identity server | Email/phone to Matrix ID mapping | GoUNITY: no server needed for verification |
| FIDO2/WebAuthn | Hardware-backed authentication | GoUNITY: persistent pseudonyms, not just auth |

---

## 9. Related components

| Component | Role | Documentation |
|:----------|:-----|:-------------|
| [GoBot](https://github.com/saschadaemgen/GoBot) | Verifies certificates in groups/communities | [Architecture](https://github.com/saschadaemgen/GoBot/blob/main/docs/ARCHITECTURE_AND_SECURITY.md) |
| [GoKey](https://github.com/saschadaemgen/SimpleGo) | Hardware verification + CRL storage | [Architecture](https://github.com/saschadaemgen/SimpleGo/blob/main/templates/gokey/docs/ARCHITECTURE_AND_SECURITY.md) |
| [GoLab](https://github.com/saschadaemgen/GoLab) | Community platform (identity consumer) | [Architecture](https://github.com/saschadaemgen/GoLab/blob/main/docs/ARCHITECTURE_AND_SECURITY.md) |

---

*GoUNITY Technical Concept v1 - April 2026*
*IT and More Systems, Recklinghausen, Germany*
