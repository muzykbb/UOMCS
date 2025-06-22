# UOMCS - Peer Discovery and Secure Session Establishment

## 1. Introduction

This document specifies the mechanisms by which UOMCS nodes discover each other and establish secure end-to-end communication sessions. While the detailed examples primarily focus on UWB as a bearer, the principles of `NodeID` advertisement and secure handshake are intended to be adaptable for UOMCS operation over other bearers via their respective Bearer Adaptation Layers (BALs).

## 2. Peer Discovery

Peer discovery is the process by which a UOMCS node becomes aware of other UOMCS nodes in its vicinity on a specific bearer.

### 2.1. UWB Bearer Peer Discovery

For the UWB bearer, discovery relies on UOMCS `BEACON` frames.

#### 2.1.1. UOMCS `BEACON` Frame (UWB)
- Each UOMCS node operating over UWB **SHOULD** periodically broadcast a `BEACON` frame via its UWB BAL.
- The `BEACON` frame **SHALL** be unencrypted at the UOMCS application level but **MUST** be signed using the broadcasting node's long-term private key (Ed25519 as per `05-security.md`). This allows recipients to verify its authenticity and integrity if they later obtain the sender's long-term public key (e.g., during a connection handshake).
- The `BEACON` frame, when transmitted over UWB, **MUST** contain at least the following UOMCS-defined data fields (which the UWB BAL will encapsulate in a UWB DLL frame):
    - `UOMCS_Protocol_Version` (e.g., 1 byte): The version of the UOMCS protocol this node primarily supports. <!-- TODO: Define versioning scheme in 08-operational-considerations.md -->
    - `Temporary_Identifier` (e.g., 4-8 bytes): To enhance privacy, nodes **SHOULD** use a temporary, randomized identifier in public beacons. This identifier helps in correlating probe requests or connection attempts before full `NodeID`s are exchanged. It **MAY** be a truncated hash of the `NodeID` or a frequently changing random value.
    - `Capabilities_Bitmask` (e.g., 2 bytes): A bitmask indicating supported UOMCS features relevant for discovery (e.g., willingness to act as a mesh router, support for IP gatewaying, support for specific UOMCS versions). <!-- TODO: Define the bitmask structure in 08-operational-considerations.md -->
    - `Timestamp` (e.g., `uint32_t` or `uint64_t`): A timestamp indicating when the beacon was generated (e.g., seconds or milliseconds since an epoch or boot time). This **SHALL** be used by recipients to assess freshness and **MAY** be used as part of an anti-replay mechanism for beacons observed over time.
    - `Signature` (e.g., 64 bytes for Ed25519): A digital signature covering the concatenated content of the UOMCS-defined fields within the `BEACON` (Version, TempID, Capabilities, Timestamp), generated using the node's long-term private key.
- The UWB BAL/DLL is responsible for transmitting these beacons on one or more well-known UWB channels/configurations.
- Nodes listening for beacons via their UWB BAL **SHALL** pass discovered peer information (including the temporary identifier, signature, and other beacon data) to the UOMCS NL or a designated discovery management entity.

#### 2.1.2. UWB Active Probing
- A node **MAY** actively probe for UOMCS peers on UWB by instructing its UWB BAL to send a `PROBE_REQ` frame.
- `PROBE_REQ` (UWB):
    - **MAY** be broadcast or directed to a known `Temporary_Identifier`.
    - **SHOULD** contain the sender's `Temporary_Identifier` and `UOMCS_Protocol_Version`.
- Nodes receiving a valid `PROBE_REQ` via their UWB BAL **SHOULD** respond with their standard UOMCS `BEACON` frame.

### 2.2. Discovery over Other Bearers
- If UOMCS operates over other bearers (e.g., Bluetooth, Wi-Fi Direct), the respective BAL **SHALL** implement a bearer-specific discovery mechanism.
- This typically involves:
    1.  Using the bearer's native discovery protocols (e.g., Bluetooth SDP/GATT advertisement, Wi-Fi Direct service discovery).
    2.  Advertising a specific UOMCS service UUID or identifier.
    3.  Exchanging UOMCS `NodeID`s or `Temporary_Identifier`s and `Capabilities` once a bearer-level connection is established or as part of the advertisement.
- The BAL for that bearer **SHALL** then use the `PeerDiscoveredCallback` (defined in `01-protocol-layers.md`) to inform the UOMCS NL.

## 3. Secure UOMCS Session Establishment

Once a peer UOMCS node is discovered (identified by at least a `Temporary_Identifier` or preferably its full `NodeID`), a secure end-to-end UOMCS session **MUST** be established before application data can be exchanged securely. This handshake provides mutual authentication based on long-term keys and derives a shared symmetric `SessionKey` for ongoing communication.

The following handshake protocol **SHALL** be used. It is typically initiated over a direct link provided by a BAL (e.g., UWB BAL).

### 3.1. Handshake Messages

The handshake involves the exchange of `CONNECT_REQ`, `CONNECT_ACK`, and `CONFIRM` messages. These are UOMCS control messages that the NL will instruct the appropriate BAL to transmit.

#### 3.1.1. `CONNECT_REQ` (Connection Request)
- **Initiator (Node A):**
    1. Generates a fresh ephemeral key pair (X25519). Let this be (ephem_privA, ephem_pubA).
    2. Constructs a `CONNECT_REQ` message containing:
        - `Initiator_NodeID` (`bytes[32]`): Node A's full UOMCS `NodeID`.
        - `Initiator_Ephemeral_PublicKey` (`bytes[32]` for X25519): ephem_pubA.
        - `Initiator_Timestamp_or_Nonce` (`uint64_t`): A current timestamp or a large random nonce to prevent replay.
        - `Supported_Cipher_Suites` (Variable, Optional): List of AEAD cipher suites Node A supports (e.g., a code for AES-256-GCM). Default is AES-256-GCM.
        - `UOMCS_Protocol_Version` (as in Beacon).
    3. Signs the entire `CONNECT_REQ` content (excluding the signature field itself) using Node A's long-term private key (Ed25519). The signature is appended.

#### 3.1.2. `CONNECT_ACK` (Connection Acknowledge)
- **Responder (Node B), upon receiving and validating `CONNECT_REQ`:**
    1. Verifies `Initiator_Timestamp_or_Nonce` against replay (see `05-security.md`).
    2. Verifies the signature on `CONNECT_REQ` using `Initiator_NodeID` (by fetching the corresponding long-term public key). If verification fails, the process **MUST** terminate.
    3. If valid, Node B generates its own fresh ephemeral key pair (X25519): (ephem_privB, ephem_pubB).
    4. Constructs a `CONNECT_ACK` message containing:
        - `Responder_NodeID` (`bytes[32]`): Node B's full UOMCS `NodeID`.
        - `Responder_Ephemeral_PublicKey` (`bytes[32]` for X25519): ephem_pubB.
        - `Responder_Timestamp_or_Nonce` (`uint64_t`): A current timestamp or nonce.
        - `Selected_Cipher_Suite` (Optional, if choices were offered): The cipher suite selected by Node B from A's list. Default is AES-256-GCM.
        - Echo of `Initiator_Timestamp_or_Nonce` (Optional, for stronger binding).
    5. Signs the entire `CONNECT_ACK` content using Node B's long-term private key (Ed25519). The signature is appended.

#### 3.1.3. `CONFIRM` (Confirmation)
- This phase verifies that both parties have derived the same session key. It typically involves two messages, one from A to B and one from B to A, or a combined exchange.
- **Message Content (Example: `A_TO_B_CONFIRM`):**
    - `Proof_A` (e.g., `bytes[32]`): A cryptographic proof, e.g., HMAC-SHA256 (using a key derived from the new `SessionKey`) over a transcript of the handshake messages or key material.
- These `CONFIRM` messages **MUST** be the first messages encrypted and authenticated with the newly derived `SessionKey`.

### 3.2. Handshake Flow

1.  **Initiation (Node A):**
    - Node A sends the signed `CONNECT_REQ` to Node B's `Temporary_Identifier` or `NodeID` (if known) via the appropriate BAL.
2.  **Response & Key Derivation (Node B):**
    - Node B receives `CONNECT_REQ`. Validates it (timestamp/nonce, signature).
    - If valid, Node B derives the ECDH shared secret: `shared_secret = X25519(ephem_privB, ephem_pubA)`.
    - Node B derives the `SessionKey` (and other necessary keys like MAC keys for confirmation if not using AEAD for it) from `shared_secret` using the KDF specified in `05-security.md` (HKDF-SHA256). The KDF `info` string **SHOULD** include `Initiator_NodeID`, `Responder_NodeID`, and nonces/public keys from the handshake to ensure uniqueness.
    - Node B sends the signed `CONNECT_ACK` back to Node A.
3.  **Key Derivation & Confirmation (Node A):**
    - Node A receives `CONNECT_ACK`. Validates it (timestamp/nonce, signature).
    - If valid, Node A derives the ECDH shared secret: `shared_secret = X25519(ephem_privA, ephem_pubB)`. This **MUST** be identical to Node B's.
    - Node A derives the `SessionKey` (etc.) using the same KDF and parameters as Node B.
    - Node A constructs and sends `A_TO_B_CONFIRM` encrypted and authenticated with the new `SessionKey`.
4.  **Confirmation (Node B):**
    - Node B receives `A_TO_B_CONFIRM`. Attempts to decrypt and verify it using its derived `SessionKey`.
    - If successful, Node B constructs and sends `B_TO_A_CONFIRM` (or its part of a combined confirmation) encrypted and authenticated with the `SessionKey`.
5.  **Session Active (Node A):**
    - Node A receives `B_TO_A_CONFIRM`. Attempts to decrypt and verify it.
    - If successful, the UOMCS session is now active. Both nodes **MUST** securely discard their ephemeral private keys (ephem_privA, ephem_privB) to ensure Perfect Forward Secrecy.

### 3.3. Handshake State Machine & Timeouts
- Each node involved in a handshake **SHALL** maintain a state for that handshake attempt (e.g., `IDLE`, `CONNECT_REQ_SENT`, `CONNECT_ACK_SENT`, `AWAIT_CONFIRM`, `ESTABLISHED`, `FAILED`).
- Timeouts **MUST** be implemented for each stage where a response is expected:
    - Initiator A waits for `CONNECT_ACK` for `TIMEOUT_CONNECT_ACK`.
    - Responder B (after sending `CONNECT_ACK`) waits for `A_TO_B_CONFIRM` for `TIMEOUT_CONFIRM_AB`.
    - Initiator A (after sending `A_TO_B_CONFIRM`) waits for `B_TO_A_CONFIRM` for `TIMEOUT_CONFIRM_BA`.
- If a timeout occurs, the handshake attempt **MUST** be considered failed for that peer, any derived key material **MUST** be discarded, and the state transitioned to `FAILED`. The node **MAY** attempt a new handshake after a backoff period.
- If a signature validation fails at any point, or if `CONFIRM` message validation fails, the handshake **MUST** be considered failed, keys discarded, and state moved to `FAILED`.
<!-- TODO: Provide specific recommended values for TIMEOUT_CONNECT_ACK, TIMEOUT_CONFIRM_AB, TIMEOUT_CONFIRM_BA. E.g., a few seconds for UWB direct link. -->
<!-- TODO: Specify exact byte-level formats for CONNECT_REQ, CONNECT_ACK, and CONFIRM messages, including field sizes and endianness. -->

## 4. Secure Ranging (UWB Specific)

While distinct from UOMCS session establishment, secure ranging is a key UWB capability.
- UOMCS nodes using UWB **MAY** initiate secure ranging sessions as defined by IEEE 802.15.4z / FiRa specifications.
- The keys used for secure ranging **SHOULD** be derived independently of the UOMCS `SessionKey`s, though they might be bootstrapped or authorized via a UOMCS session.
- The results of secure ranging (distance measurements) **MAY** be exchanged as an application payload (`Payload Type 0x02`, see `04-data-payloads.md`) over an established secure UOMCS session.

## 5. Post-Connection

Once a UOMCS session is established:
- All subsequent data between these two nodes for this session context **MUST** be encrypted and authenticated using the derived `SessionKey` and the agreed-upon AEAD cipher (AES-256-GCM).
- If communication is multi-hop, this `SessionKey` provides end-to-end security between the original sender and final recipient `NodeID`s, even if routed through intermediate nodes.