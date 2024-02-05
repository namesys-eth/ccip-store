---
eip:
title: "Off-Chain Data Write Protocol"
description: Update to Cross-Chain Write Deferral Protocol (EIP-5559) incorporating secure write deferrals to centralised databases and decentralised & mutable storages
author: (@sshmatrix), (@0xc0de4c0ffee)
discussions-to:
status: Draft
type: Standards Track
category: ERC
created:
requires:
---

## Abstract
The following proposal is an update to EIP-5559: Off-Chain Write Deferral Protocol, targeting a wider set of storage types and introducing security measures to consider for secure off-chain write deferral and retrieval. While EIP-5559 is limited to deferring write operations to L2 EVM chains and centralised databases, methods in this document enable secure write deferral to generic decentralised storages - mutable or immutable - such as IPFS, Arweave, Swarm etc. This draft alongside EIP-3668 and EIP-5559 is a significant step toward a complete and secure infrastructure for off-chain data retrieval and write deferral.

## Motivation
EIP-3668, or 'CCIP-Read' in short, has been key to retrieving off-chain data for a variety of contracts on Ethereum blockchain, ranging from price feeds for DeFi contracts, to more recently records for ENS users. The latter case is more interesting since it dedicatedly uses off-chain storage to bypass the usually high gas fees associated with on-chain storage; this aspect has a plethora of use cases well beyond ENS records and a potential for significant impact on universal affordability and accessibility of Ethereum. 

Off-chain data retrieval through EIP-3668 is a relatively simpler task since it assumes that all relevant data originating from off-chain storages is translated by CCIP-Read-compliant HTTP gateways; this includes L2 chains, centralised databases or decentralised storages. On the flip side however, so far each service leveraging CCIP-Read must handle two main tasks externally:

- writing this data securely to these storage types on their own, and

- incorporating reasonable security measures in their CCIP-Read compatible contracts for verifying this data before performing on-chain read or write operations.

Writing to a variety of centralised and decentralised storages is a broader objective compared to CCIP-Read largely due to two reasons:

1. Each storage provider typically has its own architecture that the write operation must comply with, e.g. they may require additional credentials and configuration to able to write data to them, and

2. Each storage must incorporate some form of security measures during write operations so that off-chain data's integrity can be verified by CCIP-Read contracts during data retrieval stage.

EIP-5559 was the first step toward such a tolerant 'CCIP-Write' protocol which outlined how write deferrals could be made to L2 and centralised databases. The cases of L2 and database are similar; deferral to an L2 involves routing the `eth_call` to L2, while deferral to a database can be made by extracting `eth_sign` from `eth_call` and posting the resulting signature along with the data for later verification. In both cases, no pre-flight information needs to be processed by the client and arguments of `eth_call` and `eth_sign` as specified in EIP-5559 are sufficient. This proposal extends the previous attempt by including secure write deferrals to decentralised storages, especially those which - beyond the arguments of `eth_call` and `eth_sign` - require additional pre-flight metadata from clients to successfully host users' data on their favourite storage. This document also enables more complex and generic use-cases of databases such as those which do not store the signers' addressess on chain as presumed in EIP-5559.

### Curious Case of Decentralised Storages
Decentralised storages powered by cryptographic protocols are unique in their diversity of architectures compared to centralised databases or L2 chains, both of which have canonical architectures in place. For instance, write calls to L2 chains can be generalised through the use of `ChainId` for any given `callData`; write deferral in this case is as simple as routing the `eth_call` to another contract on an L2 chain. There is no need to incorporate any additional security requirement(s) since the L2 chain ensures data integrity locally, while the global integrity can be proven by employing a state verifier scheme (e.g. EVM-Gateway) during CCIP-Read calls. Centralised databases have a very similar architecture where instead of invoking `eth_call`, the result of `eth_sign` needs to be posted on the database along with the `callData` for integrity verification by CCIP-Read.

Decentralised storages on the other hand, do not typically have EVM- or database-like environments and may have their own unique content addressing requirements. For example, IPFS, Arweave, Swarm etc all have unique content identification schemes as well as their own specific fine-tunings and/or choices of cryptographic primitives, besides supporting their own cryptographically secured namespaces. This significant and diverse deviation from EVM-like architecture results in an equally diverse set of requirements during both the write deferral operation as well as the subsequent state verifying stage.

For example, consider a scenario where the choice of storage is IPNS or ArNS. In precise terms, IPNS storage refers to immutable IPFS content wrapped in mutable IPNS namespace, which eventually serves as the reference coordinate for off-chain data. The case of ArNS is similar; ArNS is immutable Arweave content wrapped in mutable ArNS namespace. To write to IPNS or ArNS storage, the client requires more information than only the gateway URL `url` responsible for write operations and arguments of `eth_sign`. More precisely, the client must at least prompt the user for their IPNS or ArNS signature which is necessary for updating the namespaced storage. The client may also require additional information from the user such as specific arguments required by IPNS or ArNS signature. One such example is the requirement of integer `sequence` of IPNS update which goes into the construction of signature message payload. These additional user-centric requirements are not accommodated by EIP-5559, and the resolution of these issues -- among others such as batch writing -- is detailed in the following attempt towards a suitable CCIP-Write specification.

## Specification    
### Overview
The following specification revolves around the structure and description of an arbitrary off-chain storage handler tasked with the responsibility of writing to an arbitrary storage. First introduced in EIP-5559, the protocol outlined herein expands the capabilities of the `StorageHandledBy__()` revert to accept decentralised and namespaced storages. In addition, this draft proposes that besides `StorageHandledByL2()` and `StorageHandledByOffChainDatabase()`, new `StorageHandledBy__()` reverts be allowed through a publicly curated listed where each new `StorageHandledBy__()` storage handler must be accompanied by a complete documentation of its interface and design. Some foreseen examples of new storage handlers include `StorageHandledByIPFS()` for IPFS, `StorageHandledByIPNS()` for IPNS, `StorageHandledByArweave()` for Arweave, `StorageHandledByArNS()` for ArNS, `StorageHandledBySwarm()` for Swarm etc.

![](https://raw.githubusercontent.com/namesys-eth/namesys-ccip-write/main/images/schematic.png)

Similar to EIP-5559, a CCIP-Write deferral call to an arbitrary function `setValue(bytes32 key, bytes32 value)` can be described in pseudo-code as follows:

```solidity
// Define Revert Event
error StorageHandledBy__(...)

// Generic function in a contract
function setValue(
    bytes32 key, 
    bytes32 value
) external {
    // Get all necessary metadata from contract
    // Should typically contain coordinates to user's data
    config onChainConfig = getInfoFromContract(...);
    // Defer write call with relevant on-chain information
    revert StorageHandledBy__(onChainConfig, ...);
}
```

where, the following structure for `StorageHandledBy__()` must be followed:

```solidity
// Details of Revert Event
error StorageHandledBy__(
    bytes msg.sender, // Sender of call
    bytes callData, // Payload to store
    config onChainConfig // Send all necessary data from contract
);
```

#### Config 
The type `config` captures all the relevant information that the client may require from the contract to update a user's data on their favourite storage. For instance, `config` should contain the public coordinates to the user's data if such data exists on chain. In case of `StorageHandledByIPNS()` for example, `config` may contain the public key of a user's IPNS container that has been stored on chain; the case of ArNS is similar. In case of `StorageHandledByOffChainDatabase()`, `config` may contain the custom gateway URL serving a user's data or some form of unique identifier required by the client to locate the user's data. It follows that each storage handler `StorageHandledBy__()` must define the precise construction of their chosen `config` in their documentation.

```solidity
// Config Type
type config = [
        bytes[] | string[], 
        bytes[] | string[] | address[],
        ...
    ]; 
// Data inside config
config onChainConfig = [
        coordinates | [], // List of coordinates (must exist on chain), e.g. ChainId for L2, URL or identifier for off-chain storage, public key for off-chain namespaced & decentralised storages etc
        authorities | [], // List of addresses of authorities (if they exist on chain), e.g. contract address for L2, custom on-chain signer for other storages etc
        ...
    ];
```

### L2 Handler
A mimimal L2 handler only requires the list of `ChainId` values and the corresponding `contract` addresses and `StorageHandledByL2()` as defined in EIP-5559 is sufficient. In context of this proposal, `ChainId` and `contract` must be part of the `config`. There may however arise a situation where a service first stores some data on L2 and then writes - asynchronously or otherwise - to another off-chain storage type; in such cases, `config` may additionally contain the necessary metadata to write to off-chain storage.

#### EXAMPLE
```solidity
config onChainConfig = [
        [
            "11",
            "23",
            ...
        ], // ChainId values to write to
        [
            "0xc0ffee254729296a45a3885639AC7E10F9d54979",
            "0x75b6B7CEE3719850d344f65b24Db4B7433Ca6ee4",
            ...
        ], // Contract addresses on chains
        ...
    ];
```

The deferral in this case will prompt the client to submit the transaction to the relevant L2 as prescribed by the incoming `config`.

### Database Handler
A minimal database handler is similar to an L2 in the sense that:

  a) it requires the coordinates in form of gateway `urls` responsible for handling off-chain write operations (similar to `ChainId`), and

  b) it should require `eth_sign` output to secure the data and the client must prompt the users for these signatures (similar to `eth_call`).

In this case, the `config` consists of the bespoke `urls` as `coordinates`, and the addresses of `signers` (of `eth_sign`) take the place of `authorities`. The client must make sure that the signatures forwarded to the gateways match the addresses in `authorities`. If some gateways don't implement signatures, then clients could choose not to support those service providers since off-chain read-in without cryptographic verification methods is unsafe practise; there may however be exceptions to this due to which `authorities` are allowed to be empty in this proposal.

#### EXAMPLE
```solidity
config onChainConfig = [
        [
            "https://api.service.net",
            "wss://service.write.com",
            ...
        ], // URLs or other identifiers
        [
            "0xc0cac0254729296a45a3885639AC7E10F9d54979",
            "",
            "0xcafec0laE3719850d344f65b24Db4B7433Ca6ee4",
            ...
        ], // Custom signers (if they exist on chain)
        ...
    ];
```

In the above example, the client must prompt the user for a signature corresponding to each non-empty value in `authorities`, verify that the signature matches the value in `authorities` and pass the resulting signature to the respective gateway URL.

### Decentralised Storage Handler
Decentralised storages are the extremest in the sense that they come both in immutable and mutable form; the **immutable** forms locate the data through immutable content identifiers (CIDs) while **mutable** forms utilise some sort of namespace which can statically reference any dynamic content. Examples of the former include raw content hosted on IPFS and Arweave while the latter forms use IPNS and ArNS namespaces respectively to reference the raw and dynamic content. 

The case of immutable forms is similar to a database although these forms are not as useful in practise so far. This is due to the difficulty associated with posting the unique CID on chain each time a storage update is made. One way to bypass this difficulty is by storing the CID cheaply in an L2 contract; this method requires the client to update the data on both the decentralised storage as well as the L2 contract by means of two independent deferrals. CCIP-Read in this case is also expected to read from two storages to be able to fully handle a read call. Contrary to this tedious flow, namespaces can alternatively be used to statically fetch immutable CIDs. IPNS and ArNS public keys for example, can be used to refer to IPFS and Arweave CIDs respectively; this method doesn't require dual deferrals by CCIP-Write or -Read and the IPNS or Arweave public key needs to be stored on chain only once. However, accessing the IPNS and ArNS content now requires that the client must prompt the user for additional information (via `triggers`) such as IPNS and ArNS signatures in order to update the data.

Decentralised storage handlers are therefore bestowed with the ability to revert with additional `triggers` which the clients must process before calling the gateway with the results of processed triggers. This feature is not supported by EIP-5559 and services using EIP-5559 are thus incapable of storing data on decentralised namespaced & mutable storages.

#### EXAMPLE
```solidity

```

### Events
1. A public library must be maintained where each new storage handler must register their `StorageHandledBy__()` identifier. This library must exist in public domain and it should be the sole accepted standard for CCIP-Write infrastructure providers similar to [multiformats & multicodec](https://github.com/multiformats/multicodec) tables.

2. Each `StorageHandledBy__()` provider should be supported with detailed docs of their infrastructure along with a Protocol Improvement Proposal.

### Interface
`TBA`

## Backwards Compatibility
`TBA`

## Security Considerations
`TBA`

## Copyright
Copyright and related rights waived via [CC0](../LICENSE.md).