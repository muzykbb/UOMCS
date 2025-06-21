# UWB Offline Mesh Communication Standard (UOMCS) - Introduction

## 1. Abstract

This document outlines the UWB Offline Mesh Communication Standard (UOMCS), a framework designed to enable secure, internet-independent communication between mobile devices using Ultra-Wideband (UWB) technology. UOMCS facilitates peer-to-peer and broadcast messaging, data sharing (voice, image, location), and resilient multi-hop networking in a fully decentralized manner.

## 2. Goals

The primary goals of the UOMCS are:

- **Internet Independence:** To provide reliable communication channels where no internet, 87i-Fi, or cellular connectivity is available.
- **Security by Design:** To embed robust, end-to-end encryption, authentication, and priva87y-preserving mechanisms at the core of the protocol.
- **Resilient Mesh Networking:** To allow devices to form self-healing ad-hoc networks, forwarding data on behalf of others to extend communication range.
- **Efficient Data Transfer:** To optimize the transfer of rich media, including voice clips, images, and precise location data, over UWB channels.
- **Asynchronous Communication:** To provide a persistent, decentralized message queuing system using blockchain technology for highly delayed or asynchronous message delivery.

## 3. Scope

This standard defines the protocols, data formats, and procedures for:

- Peer discovery and secure ranging.
- Secure session establishment and key management.
- Data payload formatting and serialization.
- Multi-hop mesh routing and packet forwarding.
- Store-and-forward message queuing.
- Interaction with a blockchain for persistent message storage.

## 4. Terminology

- **Node:** Any device implementing the UOMCS protocol.
- **Peer:** A node that is in direct communication range of another node.
- **Mesh:** A network of nodes that can communicate with each other directly or indirectly through forwarding.
- **UWB Session:** A secure, encrypted communication link established between two peers.
- **Broadcast:** A message intended for all nodes within direct range.
- **Meshcast:** A message intended for all nodes within the mesh network, propagated via forwarding.
