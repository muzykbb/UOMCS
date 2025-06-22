# UOMCS - Protocol Stack and Layer Interfaces

## 1. Introduction

The UWB Offline Mesh Communication Standard (UOMCS) is defined by a multi-layered protocol stack. This architecture promotes modularity, separation of concerns, and the ability to adapt UOMCS to various underlying communication bearer technologies. Each layer provides services to the layer above it and utilizes services from the layer below it through well-defined interfaces.

## 2. UOMCS Core Layers

The UOMCS core protocol stack consists of the Application, Transport, and Network Layers. These layers are designed to be largely independent of the specific physical transmission medium.

### 2.1. Layer 5: Application Layer (AL)

- **Function:** The AL **SHALL** define the meaning, format, and exchange protocols for the data being transmitted between user applications or services running on UOMCS nodes.
- **Responsibilities:**
    - **Payload Definition:** The AL **SHALL** specify the structure for different application-level data types, such as text messages, voice data, location information, image chunks, etc. These are defined as `Payload Types` in `04-data-payloads.md`.
    - **Application-Specific Protocols:** The AL **MAY** define higher-level commands, transaction semantics, or state machines for specific applications (e.g., file transfer initiation, location sharing requests/responses).
    - **Serialization/Deserialization:** The AL **SHALL** be responsible for encoding application-specific data structures into a byte format suitable for the Transport Layer and, conversely, for deserializing incoming byte streams from the Transport Layer into application-usable data structures.
- **Interface to Transport Layer:** The AL **SHALL** interface with the Transport Layer to send and receive application messages. This interface **SHOULD** allow specifying the destination `NodeID` and the `Payload Type`.

### 2.2. Layer 4: Transport Layer (TL)

- **Function:** The TL **SHALL** provide end-to-end message delivery services between UOMCS applications on different nodes, abstracting the complexities of multi-hop mesh networking and potential message segmentation.
- **Responsibilities:**
    - **Segmentation and Reassembly (SAR):**
        - The TL **SHALL** segment large application messages from the AL into smaller Transport Layer Protocol Data Units (TPDUs) suitable for transmission via the Network Layer.
        - The TL **SHALL** reassemble TPDUs received from the Network Layer to reconstruct the original application message for the destination AL.
        - The SAR mechanism, including TPDU format, is detailed in `04-data-payloads.md`.
    - **End-to-End Reliability (Optional):** The TL **MAY** offer an optional reliable delivery service (e.g., using acknowledgements and retransmissions at the transport level) for applications requiring guaranteed message delivery. This is distinct from any hop-by-hop reliability offered by lower layers. <!-- TODO: Define if and how TL reliability is implemented, including ACK mechanisms and flow control. If not, this responsibility falls entirely to applications or is handled by hop-by-hop mechanisms. -->
    - **Flow Control (Optional):** If reliable transport is offered, the TL **MAY** implement end-to-end flow control mechanisms to prevent a fast sender from overwhelming a slow receiver.
    - **Multiplexing/Demultiplexing (Implicit):** The `Payload Type` field defined in `04-data-payloads.md` serves as a demultiplexing key, allowing the TL to direct incoming messages to the correct application or AL handler. Explicit port numbers are not defined at the UOMCS TL unless specified as an optional extension.
- **Interface to Network Layer:** The TL **SHALL** pass TPDUs to the Network Layer for routing and forwarding, specifying the destination `NodeID`. It receives TPDUs from the Network Layer destined for the local node.

### 2.3. Layer 3: Network Layer (NL)

- **Function:** The NL **SHALL** manage multi-hop communication across the UOMCS mesh network, handling addressing, routing, and packet forwarding between UOMCS nodes.
- **Responsibilities:**
    - **Addressing:** The NL **SHALL** use the unique UOMCS `NodeID` (see `05-security.md`) for end-to-end addressing within the UOMCS mesh.
    - **Routing:** The NL **SHALL** implement the UOMCS on-demand mesh routing protocol (based on AODV principles, detailed in `03-mesh-networking.md`) to discover and maintain paths between non-adjacent nodes. This includes generating, processing, and forwarding `RREQ`, `RREP`, and `RERR` control packets.
    - **Packet Forwarding:** Nodes **SHALL** be capable of acting as intermediaries, forwarding NL packets (which encapsulate TPDUs or NL control messages) on behalf of other nodes towards their final destination `NodeID`, as determined by the routing protocol.
    - **TTL Management:** NL data packets **MUST** include a Time-To-Live (TTL) or Hop Limit field. This field **SHALL** be decremented at each hop. Packets with a TTL of zero **MUST** be discarded.
    - **Security Integration:** The NL **SHALL** ensure that forwarded application data (encapsulated within TPDUs) remains end-to-end protected by UOMCS session security (see `05-security.md`). Intermediate forwarding nodes **MUST NOT** be able to decrypt the application payload. NL control packets (like routing messages) **MAY** have their own specific security considerations (e.g., authentication, integrity protection) as defined in `03-mesh-networking.md` and `05-security.md`.
- **Interface to Bearer Adaptation Layer (BAL):** The NL **SHALL** interface with one or more Bearer Adaptation Layers to send and receive NL packets over specific underlying communication bearers. This interface is defined in Section 3.

## 3. Bearer Adaptation Layer (BAL) Interface (Logical Layer 2.5)

To support various underlying communication technologies (bearers), UOMCS defines a logical Bearer Adaptation Layer Interface. The NL interacts with specific BAL implementations. Each BAL is responsible for interfacing UOMCS with a particular bearer technology (e.g., UWB, Bluetooth, Wi-Fi Direct).

A BAL implementation for a specific bearer (e.g., "UWB BAL," "Bluetooth BAL") **SHALL** provide the following services to the UOMCS NL:

- **`SendPacket(destination_link_address, packet_data, priority)`:**
    - Instructs the BAL to transmit an NL packet (`packet_data`) to a directly reachable peer identified by a `destination_link_address` specific to that bearer (e.g., UWB short address, Bluetooth MAC address).
    - The NL will have resolved the next-hop `NodeID` (from its routing table) to a `destination_link_address` via mechanisms provided by the BAL (see Peer Management below).
    - `priority` **MAY** be used by the BAL to influence queuing or channel access.
- **`ReceivePacketCallback(source_link_address, packet_data)`:**
    - A callback invoked by the BAL when it receives an NL packet (`packet_data`) from a peer identified by `source_link_address` over its bearer. The BAL **MUST** strip all bearer-specific headers before passing `packet_data` to the NL.
- **Peer Management Services:**
    - **`DiscoverPeers()`:** Instructs the BAL to initiate its bearer-specific peer discovery mechanism.
    - **`PeerDiscoveredCallback(node_id, link_address, capabilities)`:** A callback invoked by the BAL when a new UOMCS peer is discovered on its bearer.
        - `node_id`: The UOMCS `NodeID` of the discovered peer (obtained via a bearer-specific mechanism, e.g., from UOMCS beacons or a secure handshake over the bearer).
        - `link_address`: The bearer-specific address of the peer.
        - `capabilities`: Bearer-specific capabilities or UOMCS capabilities advertised by the peer over this bearer.
    - **`PeerLostCallback(node_id, link_address)`:** A callback invoked by the BAL when a previously discovered peer is no longer reachable on this bearer.
    - **`ResolveNodeID(node_id)`:** Queries the BAL to get the current bearer-specific `link_address` for a given UOMCS `NodeID` known to be on this bearer.
- **Link Characteristic Services:**
    - **`GetMTU(link_address_or_node_id)`:** Returns the maximum size of `packet_data` that can be transmitted in a single operation over this bearer to the specified peer.
    - **`GetLinkQuality(link_address_or_node_id)`:** **MAY** provide metrics about the current quality of the link to a peer (e.g., RSSI, LQI, retransmission rate). This can be used by the NL routing protocol.

## 4. Bearer-Specific Layers (Example: UWB Data Link and Physical Layers)

Below the BAL Interface are the actual bearer-specific protocols. For UWB, these are the UOMCS UWB Data Link Layer and the underlying UWB Physical Layer.

### 4.1. UOMCS UWB Data Link Layer (DLL)

- **Function:** This layer **SHALL** implement the UOMCS BAL interface specifically for the UWB bearer. It manages direct, single-hop communication between UOMCS peers over UWB.
- **Responsibilities specific to UWB BAL:**
    - **UWB Secure Ranging:** Initiating and managing secure ranging procedures using IEEE 802.15.4z capabilities.
    - **UWB Session Management (Initial Handshake):** Handling the UWB-specific aspects of the cryptographic handshake for initial key establishment if a secure UOMCS session is being established *directly* over UWB with a new peer. This includes formatting and exchanging `CONNECT_REQ`, `CONNECT_ACK` type messages (detailed in `02-peer-discovery-and-connection.md`).
    - **UWB Frame Formatting:** Defining the UOMCS frame structure for transmission over UWB. This frame **MUST** encapsulate NL packets or UWB-specific DLL control information. It **SHALL** include UWB-specific addressing (e.g., short addresses if used), sequence numbers, and any required MAC/authentication tags (e.g., from AES-GCM if link-level encryption is applied here, or if session encryption is done at NL/TL and tag is part of NL packet).
        <!-- TODO: Define the precise byte-level structure of a UOMCS UWB frame, distinguishing between DLL control frames (Beacon, Connect_Req, ACK) and Data frames encapsulating NL packets. Include fields like `FrameType`, `SourceAddress_UWB`, `DestinationAddress_UWB`, `SequenceNumber`, `PayloadLength`, Authentication Tag. -->
    - **UWB Link Reliability:** Implementing optional hop-by-hop reliability for unicast UWB frames using acknowledgements (ACKs) and retransmissions.
        <!-- TODO: Define UWB ACK frame format and retransmission strategy. -->
    - **UWB Channel Access:** Interfacing with the UWB PHY for channel access (e.g., CSMA/CA).
    - **UWB Beaconing/Discovery:** Implementing the UOMCS `BEACON` frame transmission and reception for UWB peer discovery, and handling `PROBE_REQ`/`PROBE_RESP` as defined in `02-peer-discovery-and-connection.md`. This populates the peer information for the `PeerDiscoveredCallback`.

### 4.2. Layer 1: Physical Layer (PHY) - UWB Example

- **Specification:** For UWB, UOMCS **SHALL** leverage the standard IEEE 802.15.4z (HRP UWB) or subsequent compatible revisions.
- **Function:** Handles the physical transmission/reception of bits over the UWB radio, channel management, and underlying FiRa-based ranging mechanisms.
- **UOMCS UWB Profile:** A UOMCS UWB Profile (potentially a separate companion document) **SHOULD** specify:
    - Mandated or recommended UWB channels, pulse repetition frequencies, preamble codes, and data rates.
    - Expected PHY layer services (e.g., data transmission, CCA).

*(Other bearer technologies like Bluetooth or Wi-Fi Direct would have their own corresponding DLL and PHY layers, with a specific BAL implementation interfacing them to the UOMCS NL).*