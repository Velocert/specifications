# A JWS signature profile conformant with JAdES Baseline-B and Open Badges v3 JWT Signature Profile

## Abstract

This document specifies how to create JSON Web Signature
([JWS](https://datatracker.ietf.org/doc/html/rfc7515)) digital signatures for
[Open Badges v3.0](https://www.imsglobal.org/spec/ob/v3p0) credentials
that are simultaneously compliant with the ETSI JAdES (JSON Advanced Electronic
Signatures) Baseline-B profile as defined in [ETSI TS 119 182-1
v1.2.1](https://www.etsi.org/deliver/etsi_ts/119100_119199/11918201/01.02.01_60/ts_11918201v010201p.pdf) and the [Open Badges v3  JSON Web Token Proof Format](https://www.imsglobal.org/spec/ob/v3p0#jwt-proof).
This dual compliance enables Open Badges credentials to be secured with legally
valid electronic signatures or seals in jurisdictions recognizing ETSI standards
while maintaining interoperability with the Open Badges ecosystem.

## Status of This Document

This is a draft specification. The latest version and source can be found in the [Velocert specifications repository](https://github.com/velocert/specifications).

## Copyright and License

Copyright © 2026 Velocert. This document is licensed under the [Creative Commons Attribution-ShareAlike 4.0 International License](https://creativecommons.org/licenses/by-sa/4.0/) (CC BY-SA 4.0).

## Feedback and Contributions

Comments, feedback, and proposals for this specification are welcome. Please
submit issues or pull requests in the [specifications
repository](https://github.com/velocert/specifications).

## 1. Introduction

Open Badges v3.0 defines a JWT-based proof format for verifiable credentials.
ETSI JAdES Baseline-B defines a profile for legally-binding electronic
signatures in European contexts. This specification bridges both standards to
create credentials that satisfy both ecosystems.

### 1.1 Terminology

- **JWS**: JSON Web Signature as defined in RFC 7515
- **JWT**: JSON Web Token as defined in RFC 7519
- **JAdES**: JSON Advanced Electronic Signatures (ETSI TS 119 182-1)
- **OBv3**: Open Badges Specification v3.0
- **Baseline-B**: Basic level of JAdES signature with claimed signing time
- **Electronic Seal**: Digital signature by an organization
- **Electronic Signature**: Digital signature by an individual
- **QTSP**: Qualified Trust Service Provider (as defined in eIDAS regulation)

### 1.2 Key Words

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [RFC2119](https://www.rfc-editor.org/rfc/rfc2119).

### 1.3 Conformance

An implementation conforms to this specification if it:

1. Generates signatures that satisfy both OBv3 JWT Proof Format requirements and
JAdES Baseline-B requirements as specified in Sections 2-4, OR
2. Verifies signatures according to the requirements in Section 5

Implementations MAY support only signature generation, only signature verification, or both.

## 2. JOSE Header Construction

The JOSE Protected Header MUST contain the following parameters to satisfy both
specifications:

### 2.1 Required Parameters

| Parameter | Type        | Value                                 | Requirement                      |
| --------- | ----------- | ------------------------------------- | -------------------------------- |
| `alg`     | String      | "RS256", "ES256", ...                 | MUST (both specs)                |
| `typ`     | String      | "JWT"                                 | MUST (OBv3), OPTIONAL (JAdES)    |
| `x5c`     | Array       | Certificate chain                     | MUST (both specs)                |
| `sigT`    | String      | ISO 8601 timestamp                    | MUST (JAdES v1.1.1), not in OBv3 |
| `iat`     | NumericDate | Unix timestamp                        | MUST (JAdES v1.2.1), not in OBv3 |
| `jwk`     | Object      | JSON Web Key validating the signature | MUST (OBv3), not in JAdES        |

Note: Parameters marked as 'not in OBv3' or 'not in JAdES' are ignored by
verifiers of the respective specification.

`jwk` public key MUST match the public key from the signing X.509 certificate, which is the first certificate in the `x5c` array (`x5c[0]`).

### 2.2 Example Header

```json
{
  "alg": "RS256",
  "typ": "JWT",
  "x5c": [
    "MIIC...", 
    "MIID..."
  ],
  "sigT": "2026-01-14T10:30:00Z",
  "jwk": {
    "kty": "RSA",
    "n": "0vx7agoebG...",
    "e": "AQAB"
  }
}
```

### 2.3 JAdES Version Compatibility

This profile supports both JAdES v1.1.1 and v1.2.1:

- **JAdES v1.1.1** requires the `sigT` parameter in the JOSE header
- **JAdES v1.2.1** requires the `iat` parameter in the JOSE header

Implementations targeting JAdES v1.2.1 SHOULD use `iat`. Implementations
targeting JAdES v1.1.1 MUST use `sigT`.

For maximum compatibility, implementations MAY include both parameters. When
both are present, they SHOULD represent the same signing time (with appropriate
conversion between Unix timestamp and ISO 8601 format).

Verifiers MUST accept credentials containing either `sigT` or `iat`, or both.

## 3. JWT Payload Construction

The JWT payload MUST be a JSON object that contains the [Open Badges v3
credential claims](https://www.imsglobal.org/spec/ob/v3p0#datamodels) and
includes the following JWT claims, as defined in [OBv3 JWT Payload
Format](https://www.imsglobal.org/spec/ob/v3p0#jwt-payload-format)

### 3.1 Required JWT Claims and Mapping

| Claim | Type        | Description        | Source                      |
| ----- | ----------- | ------------------ | --------------------------- |
| `iss` | URI         | Issuer identifier  | `issuer.id` from credential |
| `sub` | URI         | Subject identifier | `credentialSubject.id`      |
| `jti` | URI         | JWT ID             | `id` from credential        |
| `nbf` | NumericDate | Not valid before   | `validFrom` timestamp       |
| `exp` | NumericDate | Expiration time    | `validUntil` timestamp      |

JAdES does not define any requirements for the payload claims.

### 3.2 Validity Period

Signatures conforming to this specification (JAdES Baseline-B) are valid only
within the validity period of the signing certificate, as defined by its
`notBefore` and `notAfter` fields, typically 1 to 3 years. Therefore, the
`validUntil` and `exp` claims MUST be present and MUST NOT indicate a time later
than the `notAfter` value of the signing certificate.

For credentials requiring longer validation periods, implementers SHOULD
consider:

- Issuing parallel JAdES-LTA signatures for archival purposes
- Maintaining issuer-hosted re-issuance services

### 3.3 Credential Embedding

The complete Open Badge credential object SHOULD be embedded in the payload. The
JWT claims (`iss`, `sub`, `jti`, `nbf`, `exp`) act as integrity checks and MUST
match the corresponding credential properties (see Section [OBv3 Claim
Validation](#53-obv3-claim-validation))

### 3.4 Example Payload Structure

```json
{
  "@context": [
    "https://www.w3.org/2018/credentials/v1",
    "https://purl.imsglobal.org/spec/ob/v3p0/context.json"
  ],
  "id": "urn:uuid:91537dba-56cb-11ec-9bae-00163e0fa50b",
  "type": ["VerifiableCredential", "OpenBadgeCredential"],
  "issuer": {
    "id": "https://example.edu/issuers/123",
    "type": "Profile",
    "name": "Example University"
  },
  "issuanceDate": "2026-01-14T10:30:00Z",
  "validFrom": "2026-01-14T10:30:00Z",
  "credentialSubject": {
    "id": "did:example:learner123",
    "type": "AchievementSubject",
    // Note: achievement object omitted for brevity
  },
  "iss": "https://example.edu/issuers/123",
  "sub": "did:example:learner123",
  "jti": "urn:uuid:91537dba-56cb-11ec-9bae-00163e0fa50b",
  "nbf": 1736850600,
  "exp": 1768386600
}
```

## 4. Signature Generation

### 4.1 Signing Process

Implementations MUST follow these steps in order:

1. **Construct JOSE Header** as specified in Section 2
2. **Construct JWT Payload** as specified in Section 3
3. **Encode Header**: `BASE64URL(UTF8(JSON.stringify(header)))`
4. **Encode Payload**: `BASE64URL(UTF8(JSON.stringify(payload)))`
5. **Create Signing Input**: `encodedHeader || "." || encodedPayload`
6. **Hash Signing Input**: Apply SHA-256 to signing input
7. **Sign Hash**: Use the signing private key to sign the digest
8. **Encode Signature**: `BASE64URL(signature)`
9. **Create Compact JWS**: `encodedHeader || "." || encodedPayload || "." || encodedSignature`

### 4.2 Algorithm Requirements

#### 4.2.1 RSA

When using RSA signatures, the `alg` header parameter MUST be set to RS256
and implementations MUST use the following parameters:

- **Signature Algorithm**: RSA with SHA-256 (RS256)
- **Minimum Key Size**: 2048 bits
- **Hash Algorithm**: SHA-256
- **Padding**: PKCS#1 v1.5

#### 4.2.2 Elliptic Curve

When using Elliptic Curve signatures, the `alg` header parameter MUST be set to
ES256 and implementations MUST use the following parameters:

- **Curve**: P-256 (secp256r1)
- **Signature Algorithm**: ECDSA with SHA-256 (ES256)
- **Hash Algorithm**: SHA-256

## 5. Signature Verification

Verifiers MUST perform the following steps:

### 5.1 JWS Verification

1. Split the Compact JWS on "." characters into three parts
2. Base64url-decode the header and parse as JSON
3. Validate header parameters per Section 2
4. Extract public key from either:
   - `jwk` parameter, OR
   - Certificate in `x5c[0]`
5. Base64url-decode the signature
6. Verify signature over `encodedHeader || "." || encodedPayload`
7. Verify that the public key in `jwk` matches the public key in the `x5c[0]` certificate

### 5.2 Certificate Validation (JAdES)

For JAdES compliance, verifiers MUST additionally:

1. Validate certificate chain in `x5c` to a trusted root
2. Check certificate revocation status as of the signing time indicated by `sigT` or `iat` (not current time)
3. Verify all certificates in the chain were valid at signing time

### 5.3 OBv3 Claim Validation

After signature verification, implementations MUST validate the following
claims. If any validation fails, the credential MUST be rejected:

1. `iss` matches `issuer.id` in credential
2. `sub` matches `credentialSubject.id` in credential
3. `jti` matches `id` in credential
4. `nbf` matches `validFrom` timestamp
5. `exp` (if present) matches `validUntil` and hasn't expired
6. For live validation: Current time is after `nbf` and before `exp` (if
present)
7. For JAdES validation: The signing time (`sigT` or `iat`) is after `nbf` and
before `exp` (if present)

## 6. Implementation Considerations

### 6.1 Certificate Requirements

Electronic seal certificates are RECOMMENDED for sealing verifiable credentials over electronic signature certificates (individual attestation).

Certificates MUST:

- Be X.509v3 certificates compliant with RFC 5280
- For advanced or qualified electronic seals: Meet requirements of the relevant trust framework (e.g., eIDAS)
- Be issued by a Certificate Authority (e.g., QTSP for qualified status in EU, or equivalent in non-EU jurisdictions)
- Include key usage extension permitting digital signatures or non-repudiation
- Be valid at the time indicated by `sigT` or `iat`

### 6.2 Timestamp Precision

The `sigT` parameter uses ISO 8601 format with second precision and UTC timezone
(format: `YYYY-MM-DDTHH:MM:SSZ`). This differs from JWT `iat`/`nbf`/`exp` which
use Unix timestamps.

### 6.3 Interoperability

For maximum interoperability:

- Include both `jwk` AND `x5c` in header
- Validate that `jwk` matches certificate public key
- Accept `typ` value of "JWT"

### 6.4 Backwards Compatibility

Signatures created per this specification:

- CAN be verified by standard OBv3 verifiers (which will ignore the additional `sigT` or `iat` parameter)
- CAN be verified by JAdES Baseline-B verifiers (which will ignore the additional `jwk` parameter)

### 6.5 Multiple Signatures

This specification defines a single JWS signature format. Credentials requiring
multiple signatures (e.g., counter-signatures, co-signatures) are outside the
scope of this profile. Implementers requiring multiple signatures SHOULD
consider upgrading to JAdES-T, LT, or LTA

## 7. Security Considerations

### 7.1 Private Key Protection

Private keys MUST be protected according to the security level required by the
use case. For qualified electronic signatures/seals, private keys MUST meet the
requirements specified in the relevant trust framework (e.g., eIDAS for EU
contexts).

### 7.2 Certificate Validation

Implementers MUST implement full certificate chain validation including:

- Chain of trust to recognized root CA
- Certificate validity periods
- Revocation checking (CRL or OCSP)

### 7.3 Algorithm Agility

While this specification recommends ES256, implementers SHOULD design systems to
support algorithm upgrades as cryptographic requirements evolve.

## 8. References

### 8.1 Normative References

- **[RFC2119](https://www.rfc-editor.org/rfc/rfc2119.html)** Bradner, S., "Key words for use in RFCs to Indicate Requirement
  Levels", BCP 14, RFC 2119, March 1997
- **[RFC5280](https://www.rfc-editor.org/rfc/rfc5280.html)** Cooper, D., et al.,
"Internet X.509 Public Key Infrastructure Certificate and Certificate Revocation
List (CRL) Profile", RFC 5280, May 2008
- **[RFC7515](https://www.rfc-editor.org/rfc/rfc7515.html)** Jones, M., Bradley, J., and N. Sakimura, "JSON Web Signature (JWS)", RFC 7515, May 2015
- **[RFC7519](https://www.rfc-editor.org/rfc/rfc7519.html)** Jones, M., Bradley, J., and N. Sakimura, "JSON Web Token (JWT)", RFC 7519, May 2015
- **[ETSI-JAdES](https://www.etsi.org/deliver/etsi_ts/119100_119199/11918201/01.02.01_60/ts_11918201v010201p.pdf)** ETSI TS 119 182-1 V1.2.1, "Electronic Signatures and Infrastructures (ESI); JAdES digital signatures", July 2024
- **[OBv3](https://www.imsglobal.org/spec/ob/v3p0)** IMS Global Learning Consortium, "Open Badges Specification v3.0", 2021

### 8.2 Informative References

- **[RFC7517](https://datatracker.ietf.org/doc/html/rfc7517)** Jones, M., "JSON Web Key (JWK)", RFC 7517, May 2015
- **[RFC7518](https://datatracker.ietf.org/doc/html/rfc7518)** Jones, M., "JSON Web Algorithms (JWA)", RFC 7518, May 2015
