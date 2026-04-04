# GoUNITY Architecture & Security

**Document version:** Season 1 | April 2026
**Base:** smallstep/certificates (step-ca) - Apache-2.0
**Copyright:** 2026 Sascha Daemgen, IT and More Systems, Recklinghausen
**License:** Apache-2.0

---

## Overview

GoUNITY is a self-hosted Ed25519 certificate authority for SimpleX Chat identity verification. It is a fork of smallstep/certificates (step-ca) with a custom web frontend, challenge-response verification, and payment integration.

| Property | Details |
|:---------|:--------|
| Base software | step-ca v0.28+ (smallstep/certificates) |
| Language | Go |
| CA algorithm | Ed25519 |
| Certificate format | X.509 with custom OID extensions |
| HSM support | YubiKey PIV, PKCS#11, Google Cloud KMS, AWS KMS |
| CRL distribution | HTTPS, CA-signed, daily sync |
| Database | BoltDB (default), Postgres, MySQL |
| API | REST over TLS |
| Deployment | systemd service or Docker container |
| Domain | id.simplego.dev |

---

## 1. Certificate lifecycle

### Issuance

```
User -> id.simplego.dev -> register (email + password + payment)
  |
  v
GoUNITY generates Ed25519 keypair for user
  |
  v
GoUNITY creates X.509 certificate:
  - Subject CN: username
  - Custom OID 1.3.6.1.4.1.XXXXX.1: username
  - Custom OID 1.3.6.1.4.1.XXXXX.2: verification level
  - Validity: 1 year
  - Key usage: digitalSignature
  - Signed by CA key (in YubiKey HSM)
  |
  v
User downloads: private key (PEM) + signed certificate (PEM)
CA never stores the private key after delivery
```

### Verification (in SimpleX group via GoBot/GoKey)

```
User sends certificate to GoBot (DM only, never in group)
  |
  v
GoKey verifies CA signature (Ed25519 verify, local, offline)
GoKey checks: expired? On CRL?
  |
  v
GoKey sends challenge: random 32-byte nonce
  |
  v
User signs nonce with private key (CLI tool or SimpleGo T-Deck)
  |
  v
GoKey verifies: signature matches public key from certificate
  -> PASS: user proven as key holder, sharing impossible
  -> FAIL: certificate was copied, access denied
```

### Revocation

```
Admin revokes certificate via GoUNITY API or web frontend
  |
  v
GoUNITY adds username to CRL
GoUNITY signs CRL with CA key
CRL published at id.simplego.dev/v1/crl
  |
  v
GoKey fetches CRL daily via HTTPS
GoKey verifies CRL signature
GoKey stores CRL in NVS Flash
GoKey checks group members against CRL
Revoked members notified and optionally removed
```

---

## 2. What step-ca provides

| Feature | Status | Notes |
|:--------|:-------|:------|
| Ed25519 certificate signing | Built-in | `--kty OKP --curve Ed25519` |
| CRL generation | Built-in | HTTPS distribution |
| Custom OID extensions | Built-in | Via certificate templates |
| YubiKey HSM integration | Built-in | PKCS#11 provider |
| OIDC login (Google, Keycloak) | Built-in | Provisioner type |
| REST API | Built-in | Certificate CRUD |
| Database backends | Built-in | BoltDB, Postgres, MySQL |
| TLS for CA server | Built-in | Self-signed or Let's Encrypt |
| Certificate templates | Built-in | Go text/template + Sprig |
| Docker deployment | Built-in | Official image |
| ACME protocol | Built-in | Automated renewal |

### What we build on top

| Feature | Status | Notes |
|:--------|:-------|:------|
| Web frontend (registration) | To build | id.simplego.dev |
| Account system | To build | Email verification, password |
| Payment integration | To build | Registration fee (anti-spam) |
| Challenge-response endpoint | To build | Nonce generation + verification |
| GoUNITY certificate template | To build | Custom OIDs for username, level |
| CRL sync for ESP32 | To build | GoKey fetches and stores in NVS |

---

## 3. Security analysis

### CA key protection

The CA signing key lives in a YubiKey HSM. Even with root access on the GoUNITY server, an attacker cannot extract the key or sign certificates without physical possession of the YubiKey.

| Threat | Protection |
|:-------|:----------|
| Server compromise | CA key in YubiKey, not on disk |
| CA key extraction | YubiKey FIPS 140-2 Level 3 |
| False certificate issuance | Requires YubiKey physical presence |
| CRL tampering | CRL signed by CA key (also in YubiKey) |
| Database theft | Certificates are public anyway; private keys not stored |

### Verification security

| Threat | Protection |
|:-------|:----------|
| Certificate sharing | Challenge-response proves key ownership |
| Certificate replay | Each challenge is a fresh random nonce |
| Replay of signed nonce | Nonce is single-use, tracked by GoKey |
| Man-in-the-middle | Verification via DM (E2E encrypted by SimpleX) |
| Offline verification bypass | GoKey holds CA public key in firmware/eFuse |

### Known weaknesses

| ID | Severity | Description | Status |
|:---|:---------|:------------|:-------|
| GU-SEC-01 | MEDIUM | CRL timing window up to 24h between revocation and enforcement | Accepted - configurable sync interval |
| GU-SEC-02 | LOW | User signing tool needed for challenge-response (not in SimpleX app) | Planned - CLI tool, web app, or SimpleGo T-Deck integration |
| GU-SEC-03 | LOW | Registration fee may exclude users in some regions | Accepted - pricing tiers planned |

---

## 4. Deployment

### With systemd

```bash
# Initialize CA
step ca init --name="GoUNITY" --dns="id.simplego.dev" --kty OKP --curve Ed25519

# Configure YubiKey HSM (production)
step ca provisioner add gounity-admin --type JWK --create

# Start
step-ca $(step path)/config/ca.json
```

### With Docker

```bash
docker run -d \
  -v gounity-data:/home/step \
  -p 443:443 \
  -e "DOCKER_STEPCA_INIT_NAME=GoUNITY" \
  -e "DOCKER_STEPCA_INIT_DNS_NAMES=id.simplego.dev" \
  ghcr.io/saschadaemgen/gounity:latest
```

---

## 5. Integration with GoBot system

```
GoUNITY (this)          GoBot (VPS)              GoKey (ESP32)
  |                       |                         |
  |-- issues cert ------->|                         |
  |                       |-- forwards to GoKey --->|
  |                       |                    verifies locally
  |                       |                    challenge-response
  |                       |                    stores in NVS
  |                       |                         |
  |-- publishes CRL ------|-- forwards CRL -------->|
  |                       |                    stores in NVS
  |                       |                    checks members
```

GoUNITY is contacted only during registration and CRL sync.
GoUNITY never knows which groups users join.
All verification is local on the ESP32.

---

*GoUNITY Architecture & Security v1 - April 2026*
*IT and More Systems, Recklinghausen, Germany*
