<p align="center">
  <strong>GoUNITY</strong>
</p>

<p align="center">
  <strong>Verified identity and certificate authority for SimpleX Chat groups.</strong><br>
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
- Group admins have no tools to build trusted communities

GoUNITY solves this by issuing **Ed25519 certificates** that bind a verified username to a cryptographic keypair. Bans are linked to the certificate, not the SimpleX profile. Rejoin with a new name? Fine - but you need a new certificate, which costs money and requires re-verification.

---

## How it works

```
Registration:
  1. User visits id.simplego.dev
  2. Creates account (email + password + payment)
  3. GoUNITY generates Ed25519 keypair
  4. GoUNITY signs certificate with CA key (in YubiKey HSM)
  5. User receives: private key + signed certificate

Verification in group:
  1. User joins GoBot-moderated SimpleX group
  2. GoBot asks for verification (via DM)
  3. User sends certificate
  4. GoBot verifies CA signature locally (offline, no GoUNITY contact)
  5. GoBot sends challenge nonce
  6. User signs nonce with private key
  7. GoBot verifies: signature matches public key in certificate
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

---

## Security

- **CA signing key** lives in a YubiKey HSM, never on disk
- **Challenge-response** prevents certificate sharing and replay
- **Offline verification** - GoBot/GoKey checks certificates without contacting GoUNITY
- **CRL distribution** via HTTPS, signed by CA, synced daily to GoKey NVS
- **Registration fee** creates economic barrier against ban evasion
- **GoUNITY never knows** which groups users join (offline verification)

---

## Part of the GoBot system

GoUNITY is the identity layer of the GoBot moderation system:

| Component | Role | Repository |
|:----------|:-----|:-----------|
| [GoBot](https://github.com/saschadaemgen/GoBot) | Proxy + command execution | GoBot repo |
| [GoKey](https://github.com/saschadaemgen/SimpleGo) | Crypto engine (ESP32-S3) | SimpleGo template |
| **GoUNITY** | **Certificate authority** | **This repo** |

See [GoBot System Architecture](https://github.com/saschadaemgen/GoBot/blob/main/docs/SYSTEM-ARCHITECTURE.md) for the full system design.

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

## License

Apache-2.0 (inherited from smallstep/certificates)

---

<p align="center">
  <i>GoUNITY is part of the <a href="https://github.com/saschadaemgen/SimpleGo">SimpleGo ecosystem</a> by IT and More Systems, Recklinghausen, Germany.</i>
</p>
