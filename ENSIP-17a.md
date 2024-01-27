---
eip: XXX1
title: "Off-Chain Data Write Protocol"
description: The 'Off-Chain Data Write Protocol' is an update to 'Cross-Chain Write Deferral Protocol' (EIP-5559) incorporating secure write deferrals to centralised databases and decentralised (and mutable) storages
author: Avneet Singh (@sshmatrix), Code Coffee (@0xc0de4c0ffee), Nick Johnson (@arachnid)
discussions-to: https://ethereum-magicians.org/t/some-discussion/99998
status: Draft
type: Standards Track
category: ERC
created: 2024-00-00
requires: 000
---

## Abstract
The following proposal is a generalised revision to EIP-5559: Off-Chain Write Deferral Protocol, targeting a wider set of storage types and introducing security measures to consider for secure off-chain write deferral and retrieval. While EIP-5559 is limited to referring write operations to L2 EVM chains and centralised databases, methods in this document enable secure write deferral to generic decentralised storages - mutable or immutable - such as IPFS, Arweave, Swarm etc. This draft alongside EIP-3668 is a significant step toward a complete and secure infrastructure for off-chain data retrieval and write deferral.

## Motivation
EIP-3668, or 'CCIP-Read' in short, has been key to retrieving off-chain data for a variety of contracts on Ethereum blockchain, ranging from price feeds for DeFi contracts, to more recently records for ENS users. The latter case is more interesting since it dedicatedly uses off-chain storage to bypass the usually high gas fees associated with on-chain storage; this aspect has a plethora of use cases well beyond ENS records and a potential for significant impact on universal affordability and accessibility of Ethereum. 

Off-chain data retrieval through EIP-3668 is a relatively simpler task since it assumes that all relevant data originating from off-chain storages is translated by CCIP-Read-compliant HTTP gateways; this includes L2 chains, centralised databases or decentralised storages. However, each service leveraging CCIP-Read must handle two main tasks externally:

**a.** Writing this data securely to these storage types on their own, and

**b.** Incorporating reasonable security measures in their CCIP-Read compatible contracts for verifying this data before performing on-chain read or write operations.

Writing to a variety of centralised and decentralised storages is a broader objective compared to CCIP-Read largely due to two reasons:

**1.** Each storage provider typically has its own architecture that the write action must comply with, i.e. each have their own specific requirements when it comes to writing data to them, and

**2.** Each storage must incorporate some form of security measures during write operations so that off-chain data's integrity can be verified by CCIP-Read contracts during data retrieval stage.

EIP-5559 was the first step toward such a tolerant 'CCIP-Write' protocol which outlined how write deferrals could be made to L2 and centralised databases. This proposal extends the previous attempt by including secure write deferrals to decentralised storages, while also updating previous specifications with securer alternatives for writing to centralised databases.

#### Curious Case of Decentralised Storages
Decentralised storages powered by cryptographic protocols are unique in their diversity of architectures compared to centralised databases or L2 chains, both of which have canonical architectures in place. For instance, write calls to L2 chains can be generalised through the use of `ChainID` since the `calldata` remains the same; write deferral in this case is as simple as routing the call to another contract on an L2 chain. There is no need to incorporate any additional security requirement(s) since the L2 chain ensures data integrity locally, while the global integrity can be proven by employing a state verifier scheme (e.g. EVM-Gateway) during CCIP-Read calls. 

Decentralised storages on the other hand, do not typically have EVM-like environments and may have their own unique content addressing requirements. For example, IPFS, Arweave, Swarm etc all have unique content identification schemes as well as their own specific fine-tunings and/or choices of cryptographic primitives, besides supporting their own cryptographically secured namespaces. This significant and diverse deviation from EVM-like architecture results in an equally diverse set of requirements during both the write deferral operation as well as the subsequent state verifying stage. The resolution of this precise issue is detailed in the following text in an attempt towards a global CCIP-Write specification.

In contrast to EIP-5559, this proposal allows for multiple storage handlers to be stacked asynchronously in arbitrary order allowing for maximal interdependence. This feature of interdependence is necessary for highly optimised protocols which employ a mix of two or more storage types at their core.

## Specification
### Overview
The specification revolves around the structure and description of an arbitrary off-chain storage handler tasked with the responsibility of writing to an arbitrary storage. Similar to CCIP-Read, CCIP-Write protocol outlined herein comprises of 2 molecular parts: `deferrer()` and `callback()`, where the `deferrer()` function reverts with the error event `StorageHandledBy__()`, which signals the deferral of write action to the off-chain handler using events.

![](https://raw.githubusercontent.com/namesys-eth/namesys-ccip-write/main/images/schematic.png)

### L1 Handler

#### Example

### L2 Handler

#### Example

### Database Handler

#### Example

### Decentralised Storage Handler

#### Example with IPFS

#### Example with Arweave

### Stacking Handlers

#### Example

### Events

### Interface

## Backwards Compatibility

## Security Considerations

###

## Copyright
Copyright and related rights waived via [CC0](../LICENSE.md).