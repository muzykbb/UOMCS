# UOMCS - Operational Considerations and General Protocol Features

## 1. Introduction

This document specifies various operational aspects, general protocol features, and system-wide parameters that are essential for creating robust, interoperable, and future-proof UOMCS implementations. These features include protocol versioning, node capability negotiation, standardized error codes, and considerations for power management and scalability.

## 2. Protocol Versioning

To ensure that nodes running different versions of the UOMCS protocol can coexist and interoperate where possible, a simple versioning scheme **SHALL** be used.

- **Version Identifier:** The `UOMCS_Protocol_Version` field, present in `BEACON` and `CONNECT_REQ` messages, **SHALL** be a `uint8_t`.
- **Versioning Scheme:**
    - The 4 most significant bits **SHALL** represent the **Major Version**. Incompatible changes that break the protocol's fundamental structure **MUST** increment the Major Version.
    - The 4 least significant bits **SHALL** represent the **Minor Version**. Backward-compatible additions of new features **SHOULD** increment the Minor Version.
    - Example: `0x12` represents Major Version 1, Minor Version 2.
- **Interoperability Rules:**
    - Nodes with different Major Versions **MUST NOT** attempt to establish a session or route traffic for each other. They are considered incompatible.
    - A node with a higher Minor Version **SHOULD** be able to communicate with a node with a lower Minor Version (of the same Major Version) by only using the features defined in the lower version.
    - During connection establishment, the two nodes **SHOULD** agree to operate using the feature set corresponding to the lower of their two Minor Versions.

## 3. Node Capabilities Exchange

Nodes **SHALL** advertise their supported features via the `Capabilities_Bitmask` field (`uint16_t`, Big Endian) present in `BEACON` frames. This allows nodes to make informed decisions about routing and connection requests.

### 3.1. `Capabilities_Bitmask` Definition

| Bit (from LSB=0) | Name                         | Description                                                                                             |
|------------------|------------------------------|---------------------------------------------------------------------------------------------------------|
| 0                | `CAP_MESH_FORWARDING`        | If set, this node is willing and able to act as a mesh router to forward packets for others.              |
| 1                | `CAP_IP_GATEWAY`             | If set, this node is an IP Gateway Node (GN) providing internet access (see `07-optional-ip-transport-and-gateway.md`). |
| 2                | `CAP_IP_TRANSPORT`           | If set, this node supports IP packet transport over UOMCS.                                              |
| 3                | `CAP_BLOCKCHAIN_INTEGRATION` | If set, this node supports the long-term asynchronous messaging feature via blockchain (see `06-optional-blockchain-integration.md`). |
| 4                | `CAP_HIGH_POWER`             | If set, this node is not severely battery constrained (e.g., mains powered) and may be preferred for routing. |
| 5                | `CAP_HELLO_MESSAGES`         | If set, this node uses optional `UOMCS_HELLO` messages for active link sensing.                         |
| 6                | `CAP_SECURE_ROUTING`         | If set, this node supports and may require signatures on routing packets (`RREP`, `RERR`).               |
| 7-15             | `Reserved`                   | Reserved for future use. **MUST** be set to 0.                                                          |

- A node **MUST** accurately represent its capabilities in this bitmask. For example, a node setting `CAP_IP_GATEWAY` **MUST** also set `CAP_MESH_FORWARDING` and `CAP_IP_TRANSPORT`.

## 4. Standardized Error Codes (Informative)

For diagnostics and application-level feedback, UOMCS implementations **MAY** use a set of standardized error codes. The mechanism for transmitting these codes (e.g., in a session teardown message or an application-level NACK) is not strictly defined in the core standard but using common codes is **RECOMMENDED**.

| Code   | Name                           | Suggested Meaning                                                                 |
|--------|--------------------------------|-----------------------------------------------------------------------------------|
| `0x01` | `E_UNSPECIFIED`                | An unknown or unspecified error occurred.                                         |
| `0x02` | `E_VERSION_INCOMPATIBLE`       | Protocol major versions are incompatible.                                         |
| `0x03` | `E_HANDSHAKE_TIMEOUT`          | Handshake failed due to a timeout waiting for a response.                           |
| `0x04` | `E_HANDSHAKE_INVALID_SIG`      | Handshake failed due to an invalid digital signature.                             |
| `0x05` | `E_HANDSHAKE_REPLAY`           | Handshake failed due to a detected replay attack (invalid nonce or timestamp).      |
| `0x10` | `E_DECRYPTION_FAILED`          | Failed to decrypt or authenticate an incoming session message (invalid key or MAC). |
| `0x20` | `E_ROUTE_NOT_FOUND`            | NL could not find a route to the destination `NodeID`.                              |
| `0x21` | `E_MESH_TTL_EXCEEDED`          | Packet dropped because its Hop Limit (TTL) reached zero.                            |
| `0x22` | `E_MESH_QUEUE_FULL`            | Packet dropped because the mesh forwarding queue was full.                            |
| `0x23` | `E_MESH_QUEUE_TIMEOUT`         | Packet dropped from mesh queue after its queuing TTL expired.                     |
| `0x30` | `E_TRANSPORT_REASSEMBLY_FAIL`  | Message reassembly failed (e.g., timeout, missing chunks).                        |
| `0x40` | `E_UNSUPPORTED_PAYLOAD_TYPE`   | The received `Payload_Type` is not supported by this node.                        |
| `0x41` | `E_UNSUPPORTED_CAPABILITY`     | A requested action requires a capability not supported by this node.              |
| `0x50` | `E_PERMISSION_DENIED`          | Action denied by policy at the application or framework level.                    |

## 5. Power Management Considerations

- UWB communication, especially active transmission and high-frequency beaconing, can be power-intensive. Implementations **SHOULD** provide mechanisms to conserve power, particularly for battery-operated devices.
- **Strategies:**
    - **Adaptive Beaconing:** Nodes **MAY** reduce their `BEACON` frequency when idle or if battery is low.
    - **Adaptive Forwarding:** A node with low battery **SHOULD** clear its `CAP_MESH_FORWARDING` bit (if it was previously set) in its beacons to signal to others that it is no longer a preferred router.
    - **Role-Based Power Saving:** Nodes acting purely as clients (not forwarding) **MAY** enter more aggressive sleep cycles, waking only periodically to listen for beacons or incoming messages.
    - **Application-Driven Activity:** The UOMCS stack **SHOULD** avoid continuous high-power operation unless actively required by a running application.

## 6. Network Scalability and Parameter Tuning

- While UOMCS is designed for ad-hoc networks, its performance will vary with network density and size.
- **Factors impacting scalability:**
    - **`RREQ` Overhead:** In dense networks, `RREQ` broadcasts can cause significant channel congestion (broadcast storm problem). The duplicate caching mechanism is the primary defense.
    - **Routing Table Size:** Nodes must have sufficient memory to store routing table entries for active routes. The size of this table may limit the practical number of concurrent destinations.
    - **Gateway Congestion:** In networks with an internet gateway, nodes physically closer to the gateway will experience higher traffic loads, potentially becoming bottlenecks.
- **Parameter Tuning:**
    - The timeout and TTL values recommended in `03-mesh-networking.md` (e.g., `ACTIVE_ROUTE_TIMEOUT_MS`, `INITIAL_TTL_RREQ`) are starting points.
    - For different network conditions (e.g., highly mobile vs. static nodes, very large vs. small networks), these parameters **MAY** be tuned to optimize performance. For example, a more stable network might use longer route lifetimes.

## 7. Interoperability and Profiles

- To facilitate interoperability, a **UOMCS Base Profile** is implicitly defined by all non-optional features in this standard (documents `00` through `05` and `08`).
- Implementations **MUST** support the Base Profile to be considered UOMCS-compliant.
- More advanced features are defined as **Optional Profiles**:
    - **IP Transport Profile:** Requires support for features in `07-optional-ip-transport-and-gateway.md`. A node indicates support via `CAP_IP_TRANSPORT`.
    - **IP Gateway Profile:** Requires support for all of the IP Transport Profile plus the Gateway-specific functions. Indicated by `CAP_IP_GATEWAY`.
    - **Blockchain Messaging Profile:** Requires support for features in `06-optional-blockchain-integration.md`. Indicated by `CAP_BLOCKCHAIN_INTEGRATION`.
- Additional profiles for other bearers (e.g., "UOMCS over Bluetooth") **MAY** be defined in separate companion documents.