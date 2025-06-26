# UOMCS - Optional Blockchain Integration for Asynchronous Communication

## 1. Purpose and Use Case

The standard UOMCS mesh queuing mechanism (`03-mesh-networking.md`) is designed for short-term interruptions in connectivity (seconds to minutes). For long-term asynchronous messaging, where a recipient node may be offline from the UOMCS mesh for extended periods (e.g., hours, days, or indefinitely), this document specifies an **OPTIONAL** mechanism using a public blockchain or distributed ledger technology (DLT) as a persistent, decentralized "message drop box."

Nodes implementing this feature **MUST** have temporary access to the internet (or a gateway node providing such access) to interact with the chosen blockchain and decentralized storage systems. This feature is intended for high-latency, high-persistence communication and is not a substitute for real-time mesh networking.

## 2. Recommended Technology Stack

While the UOMCS standard does not mandate a specific blockchain or storage solution to allow for future flexibility, this document provides a **RECOMMENDED** reference stack to ensure a baseline for interoperability among implementations that choose to support this feature.

- **Blockchain (Layer 2):** An EVM-compatible Layer 2 scaling solution such as **Arbitrum One** or **Optimism**.
    - *Justification:* These platforms provide significantly lower transaction fees and faster confirmation times than the Ethereum mainnet while inheriting its security and decentralization. Their EVM-compatibility ensures access to a mature developer ecosystem, extensive tooling (e.g., Ethers.js, Hardhat), and widespread wallet support.
- **Decentralized Storage:** **InterPlanetary File System (IPFS)**.
    - *Justification:* IPFS is a widely adopted standard for content-addressed storage, where data is located based on its cryptographic hash (Content Identifier, or CID). This provides data integrity verification by default. A robust ecosystem of public gateways and "pinning" services exists to ensure data persistence.
- **Smart Contract Interface:** A standardized smart contract interface (see Section 4) **SHALL** be used to post and retrieve message pointers.

Implementations that support blockchain integration **SHOULD** be compatible with this reference stack.

## 3. Conceptual Workflow

### 3.1. Sending a Message via Blockchain

1.  **Preparation:** The sender node (Node A) wishes to send a message to a recipient node (Node B).
2.  **Encryption:** Node A **MUST** encrypt the message payload.
    - The **RECOMMENDED** method is **ECIES** (Elliptic Curve Integrated Encryption Scheme) using Node B's long-term UOMCS public key (Ed25519 keys can be converted to Curve25519 for this purpose, or a separate X25519 key pair can be advertised). ECIES hybrid encryption works by:
        a. Generating a temporary symmetric key (e.g., AES-256).
        b. Encrypting the message payload with this symmetric key.
        c. Encrypting the symmetric key with Node B's long-term public key using ECDH.
        d. The final encrypted package contains the encrypted symmetric key and the encrypted payload.
    - This ensures only Node B's long-term private key can decrypt the message.
3.  **Data Storage (for most payloads):**
    - The encrypted payload (from step 2) **SHALL** be uploaded to IPFS.
    - Node A **SHALL** obtain the resulting IPFS CID (version 1, Base32 encoded string is **RECOMMENDED**).
    - To ensure data persistence, Node A **SHOULD** use a "pinning service" to request that the CID is kept available on the IPFS network.
4.  **Blockchain Transaction:**
    - Node A **SHALL** interact with the UOMCS Mailbox smart contract (see Section 4) deployed on the chosen blockchain.
    - Node A **SHALL** call the `postMessage` function on the contract, providing:
        - The recipient's UOMCS `NodeID` (`bytes32`).
        - The IPFS CID (`string`) pointing to the encrypted payload.
    - This transaction is sent from Node A's blockchain wallet and requires payment of a transaction fee (gas).

### 3.2. Retrieving a Message from Blockchain

1.  **Coming Online:** The recipient node (Node B) gains internet access.
2.  **Scanning the Ledger:**
    - Node B **SHALL** query the UOMCS Mailbox smart contract for `MessagePosted` events where the `recipientNodeID` field is indexed and matches its own `NodeID`.
    - This can be done efficiently using standard blockchain client libraries (e.g., Ethers.js `contract.queryFilter()`) which query a blockchain node's RPC endpoint.
3.  **Retrieval and Decryption:**
    - For each new `MessagePosted` event found, Node B:
        a. Retrieves the IPFS CID from the event log.
        b. Fetches the encrypted data from the IPFS network using the CID (e.g., via a public IPFS gateway or a local IPFS node).
        c. Decrypts the payload using its long-term UOMCS private key (the ECIES decryption process).
4.  **Display/Processing:** The decrypted message is presented to the user or processed by Node B's application. Node B **SHOULD** keep track of processed event/transaction hashes to avoid reprocessing messages.

## 4. Smart Contract Interface (Solidity)

Implementations supporting this feature **SHOULD** interact with a smart contract that implements the following Solidity interface. A canonical, community-audited deployment **MAY** be used.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.9;

/**
 * @title IUOMCSMailbox
 * @notice An interface for a decentralized, persistent message drop box for UOMCS nodes.
 *         Messages are encrypted off-chain and a pointer (e.g., an IPFS CID) is stored on-chain.
 */
interface IUOMCSMailbox {
    /**
     * @notice Emitted when a message pointer has been successfully posted to the contract.
     * @param senderWallet The blockchain address of the wallet that sent the transaction.
     * @param recipientNodeID The UOMCS NodeID of the intended recipient. This field is indexed
     *                        to allow for efficient searching by recipients.
     * @param dataCID The Content Identifier (e.g., IPFS CID) pointing to the encrypted message
     *                payload stored in a decentralized storage system.
     * @param timestamp The block timestamp when the message was posted.
     */
    event MessagePosted(
        address indexed senderWallet,
        bytes32 indexed recipientNodeID,
        string dataCID,
        uint256 timestamp
    );

    /**
     * @notice Posts a reference (CID) to an off-chain encrypted message for a UOMCS recipient.
     * @dev This function can be called by any wallet to post a message for any recipientNodeID.
     *      Authentication and authorization are handled by the encryption layer (only the
     *      recipient can decrypt) and at the application level.
     * @param recipientNodeID The UOMCS NodeID of the recipient.
     * @param dataCID The Content Identifier for the encrypted message stored off-chain.
     */
    function postMessage(bytes32 recipientNodeID, string calldata dataCID) external;
}