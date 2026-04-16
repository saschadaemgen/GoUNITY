<p align="center">
  <strong>GoUNITY</strong>
</p>

<p align="center">
  <strong>Verified identity and certificate authority for SimpleX Chat groups and GoLab communities.</strong><br>
  Ed25519 certificates with challenge-response verification. HSM-backed. Self-hosted.<br>
  Based on <a href="https://github.com/smallstep/certificates">smallstep/certificates</a> (step-ca).
</p>

<p align="center">
  <a href="LICENSE"><img src="https://img.shields.io/badge/License-Apache--2.0-blue.svg" alt="License"></a>
  <a href="#status"><img src="https://img.shields.io/badge/status-concept-yellow.svg" alt="Status"></a>
  <a href="https://github.com/saschadaemgen/SimpleGo"><img src="https://img.shields.io/badge/ecosystem-SimpleGo-green.svg" alt="SimpleGo"></a>
</p>

---

## What GoUNITY solves

SimpleX Chat has no persistent user identity. Users can create unlimited profiles with any name. This means:

- Banned users rejoin in seconds with a new profile
- There is no way to verify who someone claims to be
- Moderation bans are useless because they are tied to temporary IDs
- Group admins and community operators have no tools to build trusted communities

GoUNITY solves this by issuing **Ed25519 certificates** that bind a verified username to a cryptographic keypair. Bans are linked to the certificate, not the SimpleX profile. Rejoin with a new name? Fine - but you need a new certificate, which costs money and requires re-verification.

GoUNITY serves two ecosystems:

- **SimpleX Chat groups** moderated by [GoBot](https://github.com/saschadaemgen/GoBot) - verified members, persistent bans, role-based access
- **[GoLab](https://github.com/saschadaemgen/GoLab) communities** - pseudonymous developer profiles, project access control, reputation. GoLab uses GoUNITY certificates as DID:key identifiers for signing all community messages (posts, reactions, follows) with cryptographic proof of authorship.

---

## How it works

```
Registration:
  1. User visits id.simplego.dev
  2. Creates account (email + password + payment)
  3. GoUNITY generates Ed25519 keypair
  4. GoUNITY signs certificate with CA key (in YubiKey HSM)
  5. User receives: private key + signed certificate

Verification in SimpleX group or GoLab community:
  1. User joins GoBot-moderated group or GoLab channel
  2. GoBot asks for verification (via DM)
  3. User sends certificate
  4. GoBot/GoKey verifies CA signature locally (offline, no GoUNITY contact)
  5. GoBot/GoKey sends challenge nonce
  6. User signs nonce with private key
  7. GoBot/GoKey verifies: signature matches public key in certificate
  8. User is verified - certificate sharing impossible

Ban enforcement:
  !ban MeinPrinz -> certificate username banned
  Rejoin with new profile -> must re-verify -> "MeinPrinz" rejected
  New certificate -> new registration -> costs money
```

---

## What step-ca gives us

GoUNITY is a fork of [smallstep/certificates](https://github.com/smallstep/certificates) (step-ca) - a production-grade, open-source certificate authority written in Go with 8.1k GitHub stars.

**We get for free (not building ourselves):**
- Ed25519 certificate signing and validation
- CRL (Certificate Revocation List) generation and HTTPS distribution
- Custom X.509 OID extensions (for username, verification level)
- YubiKey / PKCS#11 HSM integration for CA signing key
- OIDC provisioner (login via Google, Keycloak, etc.)
- REST API for certificate lifecycle
- Database backends (BoltDB, Postgres, MySQL)
- Certificate templates (Go text/template syntax)
- ACME protocol support
- Docker deployment

**What we build on top:**
- Web frontend for user registration (id.simplego.dev)
- Account management (email verification, password reset)
- Payment integration (registration fee as anti-spam barrier)
- Challenge-response verification endpoint for GoBot/GoKey
- GoUNITY-specific certificate template with custom OIDs
- DID:key identifier generation for GoLab integration
- Device certificate issuance for GoKey hardware identity

---

## Certificate format

```json
{
  "subject": {"commonName": "MeinPrinz"},
  "extensions": [
    {"id": "1.3.6.1.4.1.XXXXX.1", "critical": false, "value": "MeinPrinz"},
    {"id": "1.3.6.1.4.1.XXXXX.2", "critical": false, "value": "verified"}
  ],
  "keyUsage": ["digitalSignature"],
  "extKeyUsage": ["clientAuth"],
  "notBefore": "2026-04-04T00:00:00Z",
  "notAfter": "2027-04-04T00:00:00Z"
}
```

Custom OID `1.3.6.1.4.1.XXXXX.1` = GoUNITY username
Custom OID `1.3.6.1.4.1.XXXXX.2` = verification level

### GoLab DID:key mapping

Every GoUNITY certificate maps to a W3C DID:key identifier:

```
Certificate public key (Ed25519, 32 bytes)
  -> did:key:z6Mkf5rGMoatrSj1f4CyvuHBeXJELe9RPdzo2PKGNCKVtZxP
```

This DID:key is used as the actor identifier in all GoLab ActivityStreams messages. The human-readable username ("MeinPrinz") is for display. The DID:key is for cryptographic operations and message signing.

---

## Security

- **CA signing key** lives in a YubiKey HSM, never on disk
- **Challenge-response** prevents certificate sharing and replay
- **Offline verification** - GoBot/GoKey checks certificates without contacting GoUNITY
- **CRL distribution** via HTTPS, signed by CA, synced daily to GoKey NVS
- **Registration fee** creates economic barrier against ban evasion
- **GoUNITY never knows** which groups or GoLab communities users join (offline verification)

---

## Hardware identity (GoKey integration)

GoUNITY issues device certificates for [GoKey](https://github.com/saschadaemgen/SimpleGo) ESP32-S3 hardware:

```
1. GoUNITY signs device certificate containing ESP32's eFuse Ed25519 public key
2. Device certificate linked to user's identity certificate
3. Challenge-response proves physical possession of the device
4. GoLab profiles show "hardware verified" badge
```

Device certificates are optional. They provide the strongest identity guarantee in the ecosystem - proof that a specific physical device is controlled by the certificate holder.

See [GoKey Architecture](https://github.com/saschadaemgen/SimpleGo/blob/main/templates/gokey/docs/ARCHITECTURE_AND_SECURITY.md#5-hardware-identity) for the hardware side of this integration.

---

## Part of the GoBot system and GoLab platform

GoUNITY is the identity layer shared by both systems:

| Component | Role | Repository |
|:----------|:-----|:-----------|
| [GoBot](https://github.com/saschadaemgen/GoBot) | Moderation proxy + community relay | [GoBot repo](https://github.com/saschadaemgen/GoBot) |
| [GoKey](https://github.com/saschadaemgen/SimpleGo) | Hardware crypto engine + identity anchor | [SimpleGo repo](https://github.com/saschadaemgen/SimpleGo) |
| **GoUNITY** | **Certificate authority** | **This repo** |
| [GoLab](https://github.com/saschadaemgen/GoLab) | Community platform (uses certificates for identity) | [GoLab repo](https://github.com/saschadaemgen/GoLab) |

See [GoBot System Architecture](https://github.com/saschadaemgen/GoBot/blob/main/docs/ARCHITECTURE_AND_SECURITY.md) for the full system design.

---

## Current status

| Component | Status |
|:----------|:-------|
| GoUNITY concept and architecture | Season 1 - complete |
| step-ca fork and evaluation | In progress |
| Web frontend (id.simplego.dev) | Planned (Season 4) |
| Challenge-response endpoint | Planned (Season 4) |
| GoLab DID:key integration | Planned (Season 5+) |
| GoKey device certificates | Planned (Season 5+) |

---

## Setup (planned)

```bash
git clone https://github.com/saschadaemgen/GoUNITY.git
cd GoUNITY

# Initialize CA with Ed25519
step ca init --name="GoUNITY" --dns="id.simplego.dev" --kty OKP --curve Ed25519

# Start the CA
step-ca $(step path)/config/ca.json
```

---

## Documentation

| Document | Description |
|:---------|:-----------|
| [Architecture and Security](docs/ARCHITECTURE_AND_SECURITY.md) | Technical architecture, certificate lifecycle, threat model |
| [Concept](docs/CONCEPT.md) | High-level vision, design decisions |
| [Season Index](docs/seasons/SEASON-INDEX.md) | Links to all season documentation |

### Related documentation in other repos

| Document | Description |
|:---------|:-----------|
| [GoBot System Architecture](https://github.com/saschadaemgen/GoBot/blob/main/docs/ARCHITECTURE_AND_SECURITY.md) | Full system design (GoBot + GoKey + GoUNITY + GoLab) |
| [GoKey Architecture](https://github.com/saschadaemgen/SimpleGo/blob/main/templates/gokey/docs/ARCHITECTURE_AND_SECURITY.md) | Hardware security and device certificates |
| [GoLab Architecture](https://github.com/saschadaemgen/GoLab/blob/main/docs/ARCHITECTURE_AND_SECURITY.md) | Community platform identity integration |

---

## SimpleGo ecosystem

| Project | What it does |
|:--------|:-------------|
| [SimpleGo](https://github.com/saschadaemgen/SimpleGo) | Dedicated hardware messenger on ESP32-S3 |
| [GoRelay](https://github.com/saschadaemgen/GoRelay) | Encrypted relay server (SMP + GRP) |
| [GoChat](https://github.com/saschadaemgen/GoChat) | Browser-native encrypted chat widget |
| [GoBot](https://github.com/saschadaemgen/GoBot) | Hardware-secured moderation bot |
| [GoKey](https://github.com/saschadaemgen/SimpleGo) | Hardware crypto engine for GoBot (ESP32-S3) |
| [GoUNITY](https://github.com/saschadaemgen/GoUNITY) | Certificate authority for identity verification |
| [GoLab](https://github.com/saschadaemgen/GoLab) | Privacy-first developer community platform |
| [GoShop](https://github.com/saschadaemgen/GoShop) | End-to-end encrypted e-commerce |
| [GoTube](https://github.com/saschadaemgen/GoTube) | Encrypted video platform |
| [GoBook](https://github.com/saschadaemgen/GoBook) | Encrypted publishing platform |
| [GoOS](https://github.com/saschadaemgen/GoOS) | Privacy-focused Linux (Buildroot, RK3566) |

---

## License

Apache-2.0 (inherited from smallstep/certificates)

---

<p align="center">
  <i>GoUNITY is part of the <a href="https://github.com/saschadaemgen/SimpleGo">SimpleGo ecosystem</a> by IT and More Systems, Recklinghausen, Germany.</i>
</p>

<p align="center">
  <strong>Your certificate proves your identity. Your server never knows who you are.</strong>
</p>
