# UOMCS - Optional IP Transport and Internet Gateway

## 1. Introduction and Use Case

This document specifies an **OPTIONAL** but powerful extension to the UOMCS framework that allows for the transport of standard Internet Protocol (IP) packets (both IPv4 and IPv6) over the UOMCS mesh. This capability serves two primary purposes:

1.  **Enabling IP-based Applications:** Allows standard, unmodified IP-based applications (e.g., web browsers, SSH clients, IP-based messaging apps) to run directly between nodes within the UOMCS mesh.
2.  **Internet Gatewaying:** Enables a designated UOMCS node with external internet access (a "Gateway Node") to provide shared, rudimentary internet connectivity to other nodes in the mesh.

This feature effectively allows UOMCS to act as a wireless, multi-hop "link layer" for IP traffic, creating a mobile ad-hoc network (MANET) that can be bridged to the global internet.

## 2. IP Packet Encapsulation

To transport IP packets, UOMCS defines two new `Payload_Type` values in the Transport Layer PDU (see `04-data-payloads.md`).

| Type ID (`uint8_t`) | Description     | `Chunk_Data` Format (after reassembly) |
|---------------------|-----------------|------------------------------------------|
| `0x05`              | `IPv4 Packet`   | A complete, raw IPv4 packet, including its IP header and payload. |
| `0x06`              | `IPv6 Packet`   | A complete, raw IPv6 packet, including its IP header and payload. |

- When an application on a UOMCS node generates an IP packet, the node's IP networking stack **SHALL** pass it to a UOMCS "virtual network interface."
- This interface **SHALL** hand the raw IP packet to the UOMCS Transport Layer, which will treat it as application data.
- The UOMCS TL **SHALL** then encapsulate this IP packet into one or more TPDUs with the appropriate `Payload_Type` (`0x05` or `0x06`) and use the SAR mechanism if the packet exceeds `MAX_CHUNK_DATA_SIZE`.
- The UOMCS NL then routes these TPDUs to the destination `NodeID` as usual. The entire encapsulated IP packet is protected by the end-to-end UOMCS session security.

## 3. Address Resolution (IP to NodeID)

For IP communication *within* the UOMCS mesh, nodes need a mechanism to resolve a peer's IP address to its UOMCS `NodeID`. This is analogous to ARP for IPv4 and NDP for IPv6.

UOMCS **SHALL** use a simple, unified **UOMCS Address Resolution Protocol (UARP)**.

- **UARP Message Types:** These are defined as new UOMCS `Payload_Type` values.
    - `0x30`: `UARP_REQUEST`
    - `0x31`: `UARP_REPLY`
- **`UARP_REQUEST` Payload:**
    | Field             | Size (bytes) | Data Type  | Description                                |
    |-------------------|--------------|------------|--------------------------------------------|
    | `Target_IP_Address` | 16           | `bytes[16]`| The IPv4 or IPv6 address being queried. IPv4 addresses are IPv4-mapped IPv6 addresses (`::ffff:a.b.c.d`). |
- **`UARP_REPLY` Payload:**
    | Field             | Size (bytes) | Data Type  | Description                                |
    |-------------------|--------------|------------|--------------------------------------------|
    | `Target_IP_Address` | 16           | `bytes[16]`| The IP address that was resolved.          |
    | `Target_NodeID`     | 32           | `bytes[32]`| The `NodeID` corresponding to the IP address. |
- **UARP Workflow:**
    1.  Node A wants to send an IP packet to IP address `B_IP`. It first checks its local UARP cache.
    2.  If `B_IP` is not in the cache, Node A constructs a `UARP_REQUEST` payload for `B_IP` and sends it as a UOMCS Meshcast message.
    3.  All nodes in the mesh receive the UARP request. The node that owns `B_IP` (Node B) constructs a `UARP_REPLY` and unicasts it back to Node A's `NodeID`.
    4.  Node A receives the reply, caches the (`B_IP`, `NodeID_B`) mapping, and can now send the IP packet to `NodeID_B`.
    5.  UARP cache entries **SHOULD** have a short lifetime (e.g., 5-10 minutes).

## 4. Gateway Node (GN) Functionality

A UOMCS Gateway Node provides a bridge from the private UOMCS mesh to the public internet.

### 4.1. GN Requirements
- A GN **MUST** have at least two network interfaces: a UOMCS interface and an internet-connected interface (e.g., Wi-Fi, Ethernet, Cellular).
- A GN **MUST** be capable of routing IP packets between these two interfaces.
- A GN **MUST** perform Network Address Translation (NAT) for traffic from the UOMCS mesh to the internet. This is typically Port Address Translation (PAT).
- A GN **SHOULD** run a DHCP server on its UOMCS-facing interface to automatically assign private IP addresses, a subnet mask, a default gateway address (itself), and DNS server addresses to client nodes.
- A GN **SHOULD** act as a DNS relay/proxy, forwarding DNS queries from mesh nodes to public DNS servers.

### 4.2. Gateway Discovery
- UOMCS nodes need to discover available gateways.
- A GN **SHALL** periodically send a **`GATEWAY_ADVERTISEMENT`** message as a UOMCS Meshcast.
- **`GATEWAY_ADVERTISEMENT` Payload (`Payload_Type 0x32`):**
    | Field                 | Size (bytes) | Data Type  | Description                                                                                             |
    |-----------------------|--------------|------------|---------------------------------------------------------------------------------------------------------|
    | `Gateway_NodeID`      | 32           | `bytes[32]`| The `NodeID` of this gateway.                                                                           |
    | `Gateway_IP_Address`  | 16           | `bytes[16]`| The IP address of the gateway on the UOMCS network side (the address clients should use as their default gateway). |
    | `Backhaul_Quality`    | 1            | `uint8_t`  | An indicator of the internet connection quality (e.g., `0x01`=Low/Metered, `0x02`=Medium/Cellular, `0x03`=High/Broadband). |
    | `Subnet_Prefix_Len`   | 1            | `uint8_t`  | The prefix length of the subnet served by the GN's DHCP (e.g., 24 for a /24 subnet).                       |
- Client nodes listen for these advertisements to configure their IP stack, either manually or automatically. If multiple GNs are available, the client **MAY** use the `Backhaul_Quality` metric to choose one.

### 4.3. Client Node Configuration
- A UOMCS node wishing to use internet access **SHALL** configure its IP stack upon receiving a `GATEWAY_ADVERTISEMENT`.
- It **MAY** use DHCP to obtain an IP address from the GN or use static configuration if the network policy allows.
- It **MUST** set its default IP route to point to the `Gateway_IP_Address`.

## 5. Operational Considerations for IP Transport

- **MTU (Maximum Transmission Unit):**
    - The MTU for the UOMCS virtual network interface **MUST** be carefully managed. It will be the `MAX_CHUNK_DATA_SIZE` (from `04-data-payloads.md`) minus any IP-in-IP overhead if tunneling is used.
    - A **RECOMMENDED** default MTU for the UOMCS IP interface is **1280 bytes** to be compatible with IPv6 minimums and to accommodate UOMCS overhead without excessive fragmentation.
    - Path MTU Discovery (PMTUD) **SHOULD** be supported by node IP stacks to avoid fragmentation where possible.

- **Security:**
    - Traffic between a UOMCS client and a GN is protected by the UOMCS session security.
    - Once traffic leaves the GN for the internet, it is no longer protected by UOMCS. Standard internet security protocols (TLS, VPNs, etc.) **MUST** be used by applications for true end-to-end security.
    - The GN is a critical security point. It **MUST** be a trusted node and should be appropriately hardened against attacks from both the internet and the UOMCS mesh side.

- **Power Consumption:**
    - Operating a full IP stack and constantly communicating with a gateway is significantly more power-intensive than intermittent, native UOMCS messaging.
    - This feature is best suited for nodes where power is not the primary constraint or for on-demand use rather than continuous operation.

- **Gateway as a Bottleneck:**
    - The GN is a natural bottleneck for bandwidth and a single point of failure.
    - The UOMCS routing protocol will route all internet-bound traffic towards the GN, potentially congesting nodes near the gateway.
    - Future UOMCS profiles **MAY** explore more advanced concepts like multiple active gateways, load balancing, or failover, but this is not part of the core specification.