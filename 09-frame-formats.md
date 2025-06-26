# UOMCS - Data Link Layer Frame Formats

## 1. Introduction

This document provides the definitive byte-level specification for UOMCS Data Link Layer (DLL) frames. These formats are designed for bearer technologies that support variable-sized packet transmission, with UWB being the primary reference. All multi-byte fields specified herein **SHALL** use Big Endian (Network Byte Order).

A Bearer Adaptation Layer (BAL) is responsible for encapsulating these UOMCS DLL frames within any bearer-specific physical layer framing (e.g., PHY headers, preambles, or bearer-level checksums/FCS). The structures defined below represent the payload that the UOMCS stack provides to the BAL for transmission.

## 2. Unified DLL Header

All UOMCS DLL frames **SHALL** begin with a unified 7-byte header.

| Field                   | Size (bytes) | Description                                         |
|-------------------------|--------------|-----------------------------------------------------|
| `Frame_Control`         | 2            | A bitfield defining frame type and options.         |
| `Sequence_Number`       | 1            | A sequence number for this frame.                   |
| `Destination_Address`   | 2            | Bearer-specific 16-bit address of the recipient.    |
| `Source_Address`        | 2            | Bearer-specific 16-bit address of the sender.       |
*Total Header Size: 7 bytes*

### 2.1. `Frame_Control` Field (16 bits)

The `Frame_Control` field is structured as follows:

| Bits    | Field Name           | Description                                                                                             |
|---------|----------------------|---------------------------------------------------------------------------------------------------------|
| 0-3     | `Frame_Type`         | Defines the type of frame and the structure of its payload. See Section 3.                              |
| 4       | `Security_Enabled`   | `0` = Payload is not encrypted. `1` = Payload is encrypted and authenticated using the session key.     |
| 5       | `PAN_ID_Present`     | **RESERVED**. `0` for this version of the standard. May be used in the future to support multiple co-located UOMCS networks. |
| 6       | `ACK_Request`        | `0` = No acknowledgement is required. `1` = The recipient **MUST** send an ACK frame upon successful receipt. |
| 7       | `Reserved`           | **RESERVED**. **MUST** be set to 0.                                                                     |
| 8-9     | `Frame_Version`      | `0b00` for this version of the DLL frame format.                                                        |
| 10-15   | `Reserved`           | **RESERVED**. **MUST** be set to 0.                                                                     |

### 2.2. Address Fields
- `Destination_Address` and `Source_Address` **SHALL** contain the 16-bit short addresses assigned by the UWB bearer or an equivalent short address for other bearers.
- The address `0xFFFF` is the **RESERVED** broadcast address. A frame with this destination address **MUST** be processed by all nodes that receive it. `ACK_Request` **MUST NOT** be set for broadcast frames.

### 2.3. `Sequence_Number`
- This is an 8-bit counter maintained by the sender for each peer it communicates with. It wraps from 255 back to 0.
- It is used by the receiver to detect duplicate frames and by the sender to match received ACKs to transmitted frames.

## 3. `Frame_Type` Definitions and Payload Structures

The `Frame_Type` field in the `Frame_Control` field determines the structure of the payload that follows the 7-byte Unified DLL Header.

### 3.1. Frame Type `0x1`: `DATA_SECURE`
This frame is used to transport an encrypted UOMCS Network Layer (NL) PDU.

- **`Frame_Control` Settings:** `Frame_Type`=`0x1`, `Security_Enabled`=`1`. `ACK_Request` is typically `1`.
- **Payload Structure:**

| Field                   | Size (bytes) | Description                                                                                             |
|-------------------------|--------------|---------------------------------------------------------------------------------------------------------|
| **AES-GCM Nonce (IV)**  | 12           | The 12-byte nonce used for AES-GCM encryption. **MUST** be unique for every frame encrypted with the same session key. |
| **Ciphertext**          | Variable (N) | The encrypted UOMCS NL PDU (from `03-mesh-networking.md`).                                              |
| **Authentication Tag**  | 16           | The 16-byte AES-GCM Authentication Tag.                                                                 |
*Total Payload Size: 28 + N bytes*

- **Authenticated Associated Data (AAD):** For the AES-GCM calculation, the AAD **SHALL** be the 7-byte Unified DLL Header of this frame. This binds the ciphertext to the unencrypted header, preventing manipulation.

### 3.2. Frame Type `0x2`: `ACK` (Acknowledgement)
This frame is used to acknowledge the successful receipt of a frame that had the `ACK_Request` bit set.

- **`Frame_Control` Settings:** `Frame_Type`=`0x2`, `Security_Enabled`=`0`, `ACK_Request`=`0`.
- **Payload Structure:**

| Field                   | Size (bytes) | Description                                                 |
|-------------------------|--------------|-------------------------------------------------------------|
| `Acked_Sequence_Number` | 1            | The `Sequence_Number` of the frame being acknowledged.      |

### 3.3. Frame Type `0x3`: `BEACON`
This frame is used for peer discovery, as detailed in `02-peer-discovery-and-connection.md`.

- **`Frame_Control` Settings:** `Frame_Type`=`0x3`, `Security_Enabled`=`0`, `ACK_Request`=`0`.
- **`Destination_Address`:** `0xFFFF` (Broadcast).
- **Payload Structure:** This frame's payload **SHALL** be the "UOMCS `BEACON` Frame Payload" defined in `02`, which includes the `UOMCS_Protocol_Version`, `Temporary_Identifier`, `Capabilities_Bitmask`, `Timestamp`, and `Signature`.

### 3.4. Frame Type `0x4`: `UOMCS_CONTROL`
This frame is a generic container for unencrypted UOMCS control messages, primarily the handshake messages.

- **`Frame_Control` Settings:** `Frame_Type`=`0x4`, `Security_Enabled`=`0`. `ACK_Request` is typically `1`.
- **Payload Structure:**

| Field                   | Size (bytes) | Description                                                                 |
|-------------------------|--------------|-----------------------------------------------------------------------------|
| `Control_Message_Type`  | 1            | A sub-type identifier for the control message.                              |
| `Control_Message_Payload` | Variable     | The actual control message payload.                                         |

- **`Control_Message_Type` Values:**
    - `0x01`: `CONNECT_REQ`. The `Control_Message_Payload` is the `CONNECT_REQ` structure from `02`.
    - `0x02`: `CONNECT_ACK`. The `Control_Message_Payload` is the `CONNECT_ACK` structure from `02`.
    - `0x03`: `PROBE_REQ`. The `Control_Message_Payload` is the `PROBE_REQ` structure from `02`.
    - Others may be defined in the future.

### 3.5. Frame Type `0x5`: `CONFIRM_SECURE`
This frame is used to transport the encrypted handshake confirmation messages. It is distinct from `DATA_SECURE` as its payload is not an NL PDU.

- **`Frame_Control` Settings:** `Frame_Type`=`0x5`, `Security_Enabled`=`1`. `ACK_Request` is typically `1`.
- **Payload Structure:**

| Field                   | Size (bytes) | Description                                                                                             |
|-------------------------|--------------|---------------------------------------------------------------------------------------------------------|
| **AES-GCM Nonce (IV)**  | 12           | The 12-byte nonce used for AES-GCM encryption.                                                          |
| **Ciphertext**          | 32           | The encrypted `Authenticator` payload (`Authenticator_A` or `Authenticator_B`) from `02`.               |
| **Authentication Tag**  | 16           | The 16-byte AES-GCM Authentication Tag.                                                                 |
*Total Payload Size: 60 bytes*

- **Authenticated Associated Data (AAD):** The AAD **SHALL** be the 7-byte Unified DLL Header.

### 3.6. Summary of Frame Types

| `Frame_Type` Value | Name              | Security | ACK Req. | Description                                              |
|--------------------|-------------------|----------|----------|----------------------------------------------------------|
| `0x00`             | `Reserved`        | -        | -        | Not to be used.                                          |
| `0x01`             | `DATA_SECURE`     | Yes      | Yes/No   | Carries an encrypted Network Layer PDU.                  |
| `0x02`             | `ACK`             | No       | No       | Acknowledges a received frame.                           |
| `0x3`              | `BEACON`          | No       | No       | Broadcast frame for peer discovery.                      |
| `0x4`              | `UOMCS_CONTROL`   | No       | Yes/No   | Unencrypted control messages like `CONNECT_REQ`/`ACK`.   |
| `0x5`              | `CONFIRM_SECURE`  | Yes      | Yes/No   | Encrypted handshake `CONFIRM` messages.                  |
| `0x6` - `0xF`      | `Reserved`        | -        | -        | Reserved for future standard use.                        |

## 4. Bearer-Level Considerations (UWB Example)

- **FCS/CRC:** The UOMCS stack **SHALL** assume that the underlying bearer's PHY/MAC layer appends a Frame Check Sequence (e.g., CRC-16 or CRC-32) for physical layer error detection. This FCS is not part of the UOMCS DLL frame structure defined above.
- **Addressing:** The 16-bit short addresses used in the UOMCS DLL header are typically assigned by the UWB subsystem during initialization or association. The mapping from a UOMCS `NodeID` to these short addresses for direct peers is established during the `CONNECT_REQ`/`ACK` exchange and maintained by the UWB BAL.