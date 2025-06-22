# UOMCS - Data Payload Formats and Transport Services

## 1. Introduction

This document specifies the UOMCS Transport Layer (TL) services, primarily focusing on the format of Transport Layer Protocol Data Units (TPDUs) which carry application data, the mechanism for segmentation and reassembly (SAR) of large messages, and the detailed structure of common application `Payload Types`.

## 2. Transport Layer PDU (TPDU) Structure

All Application Layer data, after AL-specific serialization, **SHALL** be encapsulated by the Transport Layer into one or more TPDUs before being passed to the Network Layer.

The TPDU **SHALL** have the following structure (all multi-byte fields are Big Endian):

| Field                  | Size (bytes) | Data Type         | Description                                                                                                   |
|------------------------|--------------|-------------------|---------------------------------------------------------------------------------------------------------------|
| `Payload_Type`         | 1            | `uint8_t`         | Defines the kind of application data in the `Chunk_Data` (see Section 4). This field **MUST** be present.       |
| `Flags_TPDU`           | 1            | `uint8_t`         | Bitmask for transport options. This field **MUST** be present. Currently defined: Bit 0 for `END_OF_MESSAGE`. |
|                        |              |                   |   `Bit 0 (LSB): END_OF_MESSAGE (EOM)`. If set, this TPDU contains the last chunk of the message.                 |
|                        |              |                   |   `Bits 1-7: Reserved`. **MUST** be set to 0 by sender, **MUST** be ignored by receiver.                         |
| `Total_Chunks` (TC)    | 2            | `uint16_t`        | Total number of chunks the original message is segmented into. For an unsegmented message, TC **SHALL** be 1. |
| `Chunk_Index` (CI)     | 2            | `uint16_t`        | Sequence number of the current chunk (0 to TC-1). For an unsegmented message (TC=1), CI **SHALL** be 0.     |
| `Message_ID` (MID)     | 4            | `uint32_t`        | Identifies a single application message. Consistent across all chunks of the same message.                      |
| `Chunk_Data_Length`    | 2            | `uint16_t`        | Length of the `Chunk_Data` field in this TPDU, in bytes.                                                        |
| `Chunk_Data`           | Variable     | `bytes[]`         | The actual byte data for the current chunk of the application payload. Max size is `MAX_CHUNK_DATA_SIZE`.     |
*Total TPDU Header Size (excluding Chunk_Data): 12 bytes*

**Note on `Flags_TPDU.EOM`:** While `Total_Chunks` and `Chunk_Index` can determine the end of a message, the `EOM` flag provides an explicit marker for the last chunk. For a message segmented into `TC` chunks, the TPDU with `Chunk_Index == TC-1` **MUST** have the `EOM` flag set. For an unsegmented message (TC=1, CI=0), the `EOM` flag **MUST** also be set.

## 3. Segmentation and Reassembly (SAR)

### 3.1. Maximum Chunk Data Size
- The `MAX_CHUNK_DATA_SIZE` for the `Chunk_Data` field in a single TPDU is determined by the effective MTU provided by the Network Layer to the Transport Layer.
- The NL MTU is, in turn, limited by the smallest MTU of any Bearer Adaptation Layer being used, minus the UOMCS NL Header size (68 bytes, as per `03-mesh-networking.md`).
- Implementations **MUST** support a minimum `MAX_CHUNK_DATA_SIZE` of at least 256 bytes. It is **RECOMMENDED** to support larger sizes (e.g., 1024 bytes) if the underlying bearers allow, to reduce segmentation overhead.
- The actual `MAX_CHUNK_DATA_SIZE` for a given path **MAY** be dynamically discovered or configured. If not, the minimum supported value **SHALL** be assumed.

### 3.2. Segmentation Process (Sender TL)
1.  When the AL provides a message larger than `MAX_CHUNK_DATA_SIZE`:
2.  The TL **SHALL** assign a unique `Message_ID (MID)` to the entire message. This MID **SHOULD** be a random `uint32_t` or a counter maintained per session/peer to ensure uniqueness within a reasonable timeframe.
3.  The TL **SHALL** calculate the `Total_Chunks (TC)` required: `TC = ceil(TotalMessageSize / MAX_CHUNK_DATA_SIZE)`.
4.  The TL **SHALL** iterate through the message, creating `TC` chunks:
    - For each chunk `i` (from 0 to TC-1):
        - Construct a TPDU.
        - Set `Payload_Type` (from AL).
        - Set `Total_Chunks` to TC.
        - Set `Chunk_Index` to `i`.
        - Set `Message_ID` to the assigned MID.
        - Copy the segment of the original message into `Chunk_Data`.
        - Set `Chunk_Data_Length` to the actual size of this chunk's data.
        - If `i == TC-1` (last chunk), set the `EOM` bit in `Flags_TPDU`.
        - Pass the TPDU to the Network Layer for transmission.

### 3.3. Reassembly Process (Receiver TL)
1.  Upon receiving a TPDU from the NL:
2.  The TL **SHALL** inspect the `Message_ID (MID)`.
3.  If this is the first chunk received for this `MID` from this source `NodeID`:
    - Create a new reassembly buffer associated with this (`Source_NodeID`, `MID`) pair.
    - Store `Total_Chunks` (TC) from this first TPDU.
    - Start a reassembly timer (`REASSEMBLY_TIMEOUT_MS`, e.g., 30,000 ms or 30 seconds) for this `MID`.
4.  If TC from the current TPDU does not match the stored TC for this `MID`, this **MAY** be treated as an error (e.g., discard all for MID).
5.  Store the `Chunk_Data` from the TPDU in the reassembly buffer at the position indicated by `Chunk_Index (CI)`. Mark this CI as received.
6.  If the `EOM` flag is set in the current TPDU, note that the last chunk has been *seen* (but not necessarily that all preceding chunks are present).
7.  If all chunks from CI=0 to CI=(TC-1) have been received for this `MID`:
    - Cancel the `REASSEMBLY_TIMEOUT_MS` for this `MID`.
    - Concatenate all `Chunk_Data` segments in order.
    - Pass the complete, reassembled message and its `Payload_Type` to the Application Layer.
    - Discard the reassembly buffer for this `MID`.
8.  **Timeout Handling:** If `REASSEMBLY_TIMEOUT_MS` expires for an `MID` before all chunks are received, the TL **MUST** discard all received chunks for that `MID` and **MAY** notify the AL of the incomplete message reception.
9.  **Duplicate Chunks:** If a TPDU with a previously received (`MID`, `CI`) is received, it **SHOULD** be silently discarded by the TL reassembly logic.

### 3.4. SAR Parameter Values (Recommended Defaults)
- `REASSEMBLY_TIMEOUT_MS`: 30,000 ms (30 seconds). This value **SHOULD** be configurable.

## 4. Application Payload Types (`Payload_Type` field)

The `Payload_Type` field in the TPDU header identifies the format of the fully reassembled `Chunk_Data`. All multi-byte integer fields within these payload structures are Big Endian, and floating-point numbers adhere to IEEE 754.

| Type ID (`uint8_t`) | Description           | Data Format within reassembled `Chunk_Data`                                       |
|---------------------|-----------------------|-----------------------------------------------------------------------------------|
| `0x01`              | Text Message          | UTF-8 encoded string.                                                             |
| `0x02`              | Location Share        | See Section 4.1.                                                                  |
| `0x03`              | Image Data            | Reassembled complete compressed image data (e.g., WebP, JPEG, AVIF). The specific image format/MIME type is determined by the application and is not part of this payload structure itself. Applications **MAY** use a separate control message or an initial chunk with metadata to signal the image type if needed. |
| `0x04`              | Voice Data            | Reassembled complete compressed audio stream data (e.g., Opus, AAC). The specific audio codec/format is determined by the application. |
| `0x05`              | IPv4 Packet           | Raw IPv4 packet, including IP header and payload. (See `07-optional-ip-transport-and-gateway.md`) |
| `0x06`              | IPv6 Packet           | Raw IPv6 packet, including IP header and payload. (See `07-optional-ip-transport-and-gateway.md`) |
| `0x10`              | Broadcast/Meshcast Text | UTF-8 encoded string. Intended for multiple recipients.                           |
| `0x20`              | Handshake Data        | UOMCS Layer 5+ handshake or application-level secure object exchange data. Use is application-defined. |
| `0xF0 - 0xFE`       | Experimental Use      | Reserved for experimental payload types.                                          |
| `0xFF`              | Reserved              | Not to be used.                                                                   |

### 4.1. Payload Type `0x02`: Location Share

This payload type is used to share geolocation information. All floating-point numbers are IEEE 754. All multi-byte integers are Big Endian.

| Field                      | Size (bytes) | Data Type         | Description                                                                                                    |
|----------------------------|--------------|-------------------|----------------------------------------------------------------------------------------------------------------|
| `Latitude`                 | 8            | `double`          | Latitude in decimal degrees (WGS84). Range: -90.0 to +90.0.                                                      |
| `Longitude`                | 8            | `double`          | Longitude in decimal degrees (WGS84). Range: -180.0 to +180.0.                                                   |
| `Timestamp_Location`       | 8            | `uint64_t`        | Milliseconds since Unix epoch (UTC) when this location fix was obtained.                                         |
| `Horizontal_Accuracy`      | 4            | `float`           | Horizontal accuracy in meters. If unknown, set to a pre-defined maximum value (e.g., `MAX_FLOAT`) or specific NaN. |
| `Flags_Location`           | 1            | `uint8_t`         | Bitmask for optional fields' presence and other info.                                                          |
|                            |              |                   |   `Bit 0: Altitude_Present`. If set, `Altitude` field is present.                                              |
|                            |              |                   |   `Bit 1: Vertical_Accuracy_Present`. If set, `Vertical_Accuracy` field is present.                            |
|                            |              |                   |   `Bit 2: Speed_Present`. If set, `Speed` field is present.                                                    |
|                            |              |                   |   `Bit 3: Bearing_Present`. If set, `Bearing` field is present.                                                |
|                            |              |                   |   `Bits 4-7: Reserved`. **MUST** be 0.                                                                         |
| `Altitude` (Optional)      | 4            | `float`           | Altitude in meters above WGS84 ellipsoid. Present if `Flags_Location.Altitude_Present` is set.                 |
| `Vertical_Accuracy`(Opt.)  | 4            | `float`           | Vertical accuracy in meters. Present if `Flags_Location.Vertical_Accuracy_Present` is set.                     |
| `Speed` (Optional)         | 4            | `float`           | Speed in meters per second over ground. Present if `Flags_Location.Speed_Present` is set.                        |
| `Bearing` (Optional)       | 2            | `uint16_t`        | Bearing (course over ground) in degrees from true north (0-359). Present if `Flags_Location.Bearing_Present` is set. |
*Total Size: 29 bytes (minimum) up to 29 + 4 + 4 + 4 + 2 = 43 bytes (maximum)*

**Processing `Location Share`:**
- The receiver **MUST** check `Flags_Location` to determine which optional fields are present and parse accordingly.
- If an optional field's corresponding "Present" flag is not set, that field's bytes are not included in the payload.

## 5. Transport Layer Reliability and Flow Control

UOMCS Transport Layer, as defined here, primarily provides segmentation and reassembly. End-to-end reliability (beyond any hop-by-hop reliability offered by BALs) and end-to-end flow control are **NOT MANDATED** at this layer in the core UOMCS specification.
- Applications requiring guaranteed delivery or sophisticated flow control **SHOULD** implement these mechanisms at the Application Layer or use a UOMCS profile that extends TL services.
- The UOMCS NL and BALs **MAY** provide mechanisms that contribute to overall reliability (e.g., DLL ACKs for UWB), but these are hop-by-hop.