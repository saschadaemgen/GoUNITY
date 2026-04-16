# GoUNITY Architecture & Security

**Document version:** Season 1 | April 2026
**Component:** GoUNITY certificate authority (identity server)
**Copyright:** 2026 Sascha Daemgen, IT and More Systems, Recklinghausen
**License:** Apache-2.0

---

## Overview

GoUNITY is a self-hosted Ed25519 certificate authority for SimpleX Chat identity verification and GoLab community identity. It is a fork of smallstep/certificates (step-ca) with a custom web frontend, challenge-response verification, and payment integration.

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

### 1.1 Issuance

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
  |
  v
Certificate public key maps to DID:key identifier:
  did:key:z6Mk... (used as actor ID in GoLab messages)
```

### 1.2 Verification (in SimpleX group or GoLab community)

```
User sends certificate to GoBot (DM only, never in group/channel)
  |
  v
GoKey verifies CA signature (Ed25519 verify, local, offline)
GoKey checks: expired? On CRL?
  |
  v
GoKey sends challenge: random 32-byte nonce
  |
  v
User signs nonce with private key (CLI tool, web app, or SimpleGo T-Deck)
  |
  v
GoKey verifies: signature matches public key from certificate
  -> PASS: user proven as key holder, sharing impossible
  -> FAIL: certificate was copied, access denied
```

### 1.3 Revocation

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
GoKey checks group members and community users against CRL
Revoked members notified and optionally removed
```

### 1.4 Device certificate issuance (GoKey hardware identity)

```
User connects GoKey ESP32-S3 device
  |
  v
GoKey generates Ed25519 keypair in eFuse (one-time, irreversible)
GoKey exports public key
  |
  v
GoUNITY signs device certificate:
  - Subject CN: "GoKey-001"
  - Custom OID 1.3.6.1.4.1.XXXXX.1: linked username
  - Custom OID 1.3.6.1.4.1.XXXXX.3: "device"
  - Custom OID 1.3.6.1.4.1.XXXXX.4: eFuse pubkey fingerprint
  - Validity: 1 year
  - Signed by CA key (in YubiKey HSM)
  |
  v
Device certificate stored on GoKey (encrypted NVS)
User identity now has hardware-backed proof of possession
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
| DID:key generation | To build | Derive DID:key from certificate pubkey |
| Device certificate workflow | To build | GoKey eFuse pubkey -> signed device cert |

---

## 3. GoLab identity integration

### 3.1 DID:key identifiers

Every GoUNITY certificate maps to a W3C DID:key identifier:

```
Ed25519 public key (32 bytes)
  -> Multicodec prefix: 0xed01 (Ed25519)
  -> Base58btc encode: z6Mk...
  -> DID:key: did:key:z6Mkf5rGMoatrSj1f4CyvuHBeXJELe9RPdzo2PKGNCKVtZxP
```

This DID:key is used as the `actor` field in all GoLab ActivityStreams messages. The mapping is deterministic - the same certificate always produces the same DID:key.

### 3.2 GoLab message signing

GoLab messages are signed with the certificate's private key:

```json
{
  "@context": "https://www.w3.org/ns/activitystreams",
  "type": "Create",
  "actor": "did:key:z6Mkf5rGMoatr...",
  "object": {"type": "Note", "content": "..."},
  "proof": {
    "type": "Ed25519Signature2020",
    "verificationMethod": "did:key:z6Mkf5rGMoatr...",
    "proofValue": "z..."
  }
}
```

Any recipient can verify the signature using only the DID:key (which contains the public key). No GoUNITY server contact needed.

### 3.3 Certificate scope

| Context | What GoUNITY provides | What GoUNITY does NOT know |
|:--------|:---------------------|:--------------------------|
| Registration | Username, email, payment | - |
| SimpleX group | Certificate for verification | Which groups user joins |
| GoLab community | Same certificate, mapped to DID:key | Which communities user joins |
| GoKey device | Device certificate linked to identity | Which device is used where |

GoUNITY is contacted only during registration, certificate renewal, and CRL sync. It never learns about the user's community activity.

---

## 4. Security analysis

### 4.1 CA key protection

The CA signing key lives in a YubiKey HSM. Even with root access on the GoUNITY server, an attacker cannot extract the key or sign certificates without physical possession of the YubiKey.

| Threat | Protection |
|:-------|:----------|
| Server compromise | CA key in YubiKey, not on disk |
| CA key extraction | YubiKey FIPS 140-2 Level 3 |
| False certificate issuance | Requires YubiKey physical presence |
| CRL tampering | CRL signed by CA key (also in YubiKey) |
| Database theft | Certificates are public anyway; private keys not stored |

### 4.2 Verification security

| Threat | Protection |
|:-------|:----------|
| Certificate sharing | Challenge-response proves key ownership |
| Certificate replay | Each challenge is a fresh random nonce |
| Replay of signed nonce | Nonce is single-use, tracked by GoKey |
| Man-in-the-middle | Verification via DM (E2E encrypted by SimpleX) |
| Offline verification bypass | GoKey holds CA public key in firmware/eFuse |
| GoLab message forgery | Every message signed with certificate key, verifiable by any recipient |

### 4.3 Known weaknesses

| ID | Severity | Description | Status |
|:---|:---------|:------------|:-------|
| GU-SEC-01 | MEDIUM | CRL timing window up to 24h between revocation and enforcement | Accepted - configurable sync interval |
| GU-SEC-02 | LOW | User signing tool needed for challenge-response (not in SimpleX app) | Planned - CLI tool, web app, or SimpleGo T-Deck |
| GU-SEC-03 | LOW | Registration fee may exclude users in some regions | Accepted - pricing tiers planned |
| GU-SEC-04 | LOW | DID:key is deterministic - same cert always same DID | By design - use context certificates for unlinkability |

---

## 5. Deployment

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

## 6. System integration

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
  |                       |                         |
  |-- issues device cert ->                         |
  |   (GoKey eFuse pubkey)                     stores device cert
  |                                            hardware identity

GoLab App Server         GoBot (relay)
  |                       |
  | (no direct contact    | verifies certificates
  |  with GoUNITY)        | enforces CRL
  |                       | checks power levels
  |                       | fans out to subscribers
```

GoUNITY is contacted only during registration, device cert issuance, and CRL sync. GoUNITY never knows which groups or GoLab communities users join. All verification is local on GoKey or GoBot.

---

## 7. Related components

| Component | Role | Documentation |
|:----------|:-----|:-------------|
| [GoBot](https://github.com/saschadaemgen/GoBot) | Moderation proxy + community relay | [Architecture](https://github.com/saschadaemgen/GoBot/blob/main/docs/ARCHITECTURE_AND_SECURITY.md) |
| [GoKey](https://github.com/saschadaemgen/SimpleGo) | Hardware crypto + device identity | [Architecture](https://github.com/saschadaemgen/SimpleGo/blob/main/templates/gokey/docs/ARCHITECTURE_AND_SECURITY.md) |
| [GoLab](https://github.com/saschadaemgen/GoLab) | Community platform (uses certificates) | [Architecture](https://github.com/saschadaemgen/GoLab/blob/main/docs/ARCHITECTURE_AND_SECURITY.md) |

---

*GoUNITY Architecture & Security v2 - April 2026*
*IT and More Systems, Recklinghausen, Germany*
