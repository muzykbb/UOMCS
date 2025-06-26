# UOMCS - Security and Privacy

## 1. Introduction

Security and privacy are fundamental pillars of the UOMCS framework, not optional add-ons. The design **SHALL** provide robust end-to-end protection for user data and **SHOULD** mitigate risks associated with metadata leakage and protocol manipulation. This document specifies the UOMCS security architecture, cryptographic primitives, key management protocols, and privacy-enhancing mechanisms.

## 2. Core Security Principles

- **Confidentiality:** Application data payload **MUST** be unintelligible to any party other than the intended recipient(s). This is achieved via strong end-to-end encryption.
- **Integrity:** Messages **MUST** be protected from tampering or modification in transit.
- **Authenticity:** Nodes **MUST** be able to verify the identity of the peers with whom they establish a session.
- **Perfect Forward Secrecy (PFS):** The compromise of a node's long-term private key **MUST NOT** compromise the confidentiality of past communication sessions.
- **Privacy:** The protocol **SHOULD** minimize the leakage of personally identifiable information (PII) and metadata that could be used for long-term tracking of users.

## 3. Cryptographic Primitives

UOMCS implementations **MUST** adhere to the following specified cryptographic algorithms to ensure interoperability and security.

- **Asymmetric Signature Algorithm:** Ed25519, as specified in [RFC8032], for signing handshake messages and beacons.
- **Asymmetric Key Exchange Algorithm:** X25519, as specified in [RFC7748], for Elliptic Curve Diffie-Hellman (ECDH) key exchange to establish a shared secret.
- **Symmetric Cipher:** AES-256 in Galois/Counter Mode (AES-256-GCM), as specified in [FIPS197] and [NIST_SP800-38D], for authenticated encryption of all post-handshake data.
- **Hash Function:** SHA-256, as specified in [FIPS180-4], for deriving `NodeID`s and for use within other cryptographic functions like HKDF.
- **Key Derivation Function (KDF):** HMAC-based Extract-and-Expand Key Derivation Function (HKDF) using SHA-256 (HKDF-SHA-256), as specified in [RFC5869].

## 4. Identity and Key Management

### 4.1. Node Identity (`NodeID`)
- Each UOMCS node **SHALL** generate a long-term key pair for the Ed25519 signature algorithm. The private key of this pair is the node's primary long-term secret and **MUST** be stored securely.
- The node's unique `NodeID` **SHALL** be the 32-byte output of the SHA-256 hash function applied to the node's 32-byte Ed25519 long-term public key. `NodeID = SHA256(Ed25519_Public_Key)`.

### 4.2. UOMCS Session Keys
- UOMCS sessions are protected by symmetric keys derived using the ECDH handshake detailed in `02-peer-discovery-and-connection.md`.
- The ECDH shared secret derived from the X25519 key exchange **SHALL** be used as the Input Keying Material (IKM) for the HKDF.

### 4.3. Key Derivation using HKDF-SHA256
The KDF **SHALL** be used to derive all necessary session keys from the ECDH shared secret.

- **HKDF Extract Step:**
    - `salt`: A salt value **SHALL** be constructed by concatenating `Initiator_Nonce` and `Responder_Nonce` from the handshake messages (`CONNECT_REQ`, `CONNECT_ACK`). `salt = Initiator_Nonce || Responder_Nonce`. This provides a unique, non-secret salt for each handshake.
    - `IKM (Input Keying Material)`: The 32-byte ECDH shared secret.
    - `PRK = HMAC-SHA256(salt, IKM)`
    *The result `PRK` is a Pseudorandom Key of 32 bytes.*

- **HKDF Expand Step:**
    - The `PRK` is used to derive multiple keys for different purposes. The `info` parameter provides context separation.
    - `info`: A context string **SHALL** be constructed by concatenating `Initiator_NodeID`, `Responder_NodeID`, and a fixed-string label for the key being derived.
    - The following keys **SHALL** be derived:
        1.  **`UOMCS_Session_Key_Encrypt` (32 bytes):** The AES-256 key for data encryption/decryption.
            - `info` string: `Initiator_NodeID || Responder_NodeID || "UOMCS_ENCRYPT_V1"`
        2.  **`UOMCS_Session_Key_Confirm` (32 bytes):** The HMAC-SHA256 key for generating and verifying handshake `CONFIRM` messages.
            - `info` string: `Initiator_NodeID || Responder_NodeID || "UOMCS_CONFIRM_V1"`
    - The derivation is `OKM = HMAC-SHA256(PRK, info || 0x01)`.

Implementations **MUST** follow this KDF process precisely to ensure both parties derive identical session keys.

## 5. Data Protection

### 5.1. Session Data Confidentiality and Integrity
- All UOMCS data packets (NL PDUs of type `UOMCS_DATA`) exchanged between two nodes within an established session **MUST** be protected using AES-256-GCM.
- The key used **SHALL** be `UOMCS_Session_Key_Encrypt`.
- The entire NL PDU (including the Common NL Header from `03-mesh-networking.md` and its TPDU payload) **SHALL** be treated as the plaintext to be encrypted.
- The 12-byte AES-GCM nonce (IV) **MUST** be unique for every encryption operation with the same key. It is **RECOMMENDED** that the nonce be constructed from a combination of a fixed part unique to the sender and a strictly increasing 64-bit counter part. The nonce **MUST** be transmitted along with the ciphertext. <!-- TODO: Precisely define the structure of the encrypted message, including placement of the nonce and authentication tag. This should be part of the UWB DLL Frame Format for encrypted data. -->
- The AES-GCM Authentication Tag (16 bytes) provides integrity and authenticity for the ciphertext.

### 5.2. Anti-Replay Mechanism for Session Data
- The uniqueness of the AES-GCM nonce for each message under a given key is the primary cryptographic anti-replay mechanism.
- A receiving node **MUST** ensure it does not process a message with a nonce it has already seen for that session.
- To handle node reboots or state loss, the nonce's counter part **SHOULD** be large enough (e.g., 64 bits) to make accidental reuse extremely unlikely over the lifetime of a device.
- **RECOMMENDED:** Applications or the transport layer **SHOULD** also include a message sequence number or timestamp within their data payloads as an additional, application-level defense against replay attacks, especially for idempotent operations.

## 6. Privacy Considerations

- **Identifier Rotation in Beacons:** As specified in `02-peer-discovery-and-connection.md`, the use of a `Temporary_Identifier` in UWB `BEACON` frames is **RECOMMENDED** to prevent passive long-term tracking of a device's `NodeID`. The full `NodeID` **SHALL** only be revealed during a direct, authenticated handshake.
- **User Consent:** Applications built on UOMCS **MUST** obtain explicit user consent before sharing sensitive data, particularly location information. The UOMCS framework itself does not permit surreptitious background data sharing.
- **Traffic Analysis:** While end-to-end encryption protects content, metadata such as `Source_NodeID` and `Destination_NodeID` in the NL header are visible to intermediate nodes to enable routing. This exposes communication patterns.
    - **Mitigation (Future Work):** Advanced privacy-preserving routing schemes (e.g., onion routing) are complex and not included in the UOMCS core specification but could be subjects of future research or optional profiles.
    - The use of multiple bearers and dynamic route changes may naturally provide some limited resistance to simple traffic analysis.

## 7. Security of Routing and Control Messages

- As a baseline, routing packets (`RREQ`, `RREP`, `RERR`) are not encrypted, as they must be read by intermediate nodes that are not part of the end-to-end session.
- **Denial-of-Service (DoS) on Routing:**
    - `RREQ` storms are mitigated by the duplicate caching mechanism specified in `03-mesh-networking.md`.
    - Malicious nodes could still inject bogus `RREQ`s or `RERR`s to disrupt the network.
- **Route Manipulation:** Without protection, malicious nodes could lie in `RREP`s (e.g., claim a route they don't have) or forge `RERR`s to blackhole traffic or partition the network.
- **RECOMMENDED Security Enhancement (Optional Profile):**
    - To secure routing, a UOMCS security profile **MAY** mandate that `RREP` and `RERR` messages be digitally signed.
    - A signed `RREP` would have a signature from the `Responding_NodeID` (the destination) over the RREP's key fields (`Originator_NodeID`, `Responding_Node_Seq_Num`, `Lifetime_RREP`). Intermediate nodes forwarding this `RREP` cannot forge it.
    - A signed `RERR` would be signed by the `Error_Source_NodeID`, preventing forgery of route takedowns.
    - This adds computational and bandwidth overhead but provides significantly stronger guarantees against routing attacks. The decision to implement this **MAY** be based on the application's threat model.

## 8. Implementation Security Requirements

- **Secure Key Storage:** Long-term private keys (Ed25519) **MUST** be stored securely, protected from unauthorized access or extraction. The use of hardware-backed keystores (e.g., Secure Enclave, TPM, TrustZone) is **STRONGLY RECOMMENDED**.
- **Cryptographic Library:** Implementations **MUST** use well-vetted, constant-time implementations of all cryptographic algorithms to prevent side-channel attacks (e.g., timing attacks).
- **Random Number Generation:** A cryptographically secure pseudo-random number generator (CSPRNG) **MUST** be used for generating all keys, nonces, `Temporary_Identifier`s, and other security-critical random values.
- **Revocation:** The UOMCS core protocol does not define a mechanism for real-time revocation of a compromised long-term key. If a node's key is compromised, it must be considered permanently untrusted by its `NodeID`. Revocation lists or out-of-band methods for blocklisting `NodeID`s are application-level concerns or topics for future UOMCS security profiles.