---
EIP: --
TITLE: "Off-Chain Data Write Protocol"
DESCRIPTION: Update to Cross-Chain Write Deferral Protocol (EIP-5559) incorporating secure write deferrals to centralised databases and decentralised & mutable storages
AUTHOR: (`@sshmatrix`), (`@0xc0de4c0ffee`), (`@arachnid`)
DISCUSSIONS-TO: --
STATUS: Draft
TYPE: Standards Track
CATEGORY: ERC
CREATED: --
REQUIRES: --
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

For example, consider a scenario where the choice of storage is IPNS or ArNS. In precise terms, IPNS storage refers to immutable IPFS content wrapped in mutable IPNS namespace, which eventually serves as the reference coordinate for off-chain data. The case of ArNS is similar; ArNS is immutable Arweave content wrapped in mutable ArNS namespace. To write to IPNS or ArNS storage, the client requires more information than only the gateway URL responsible for write operations and arguments of `eth_sign`. More precisely, the client must at least prompt the user for their IPNS or ArNS signature which is necessary for updating the namespaced storage. The client may also require additional information from the user such as specific arguments required by IPNS or ArNS signature. One such example is the requirement of integer `sequence` of IPNS update which goes into the construction of signature message payload. These additional user-centric requirements are not accommodated by EIP-5559, and the resolution of these issues - among others such as batch writing - is detailed in the following attempt towards a suitable CCIP-Write specification.

## Specification    
### Overview
The following specification revolves around the structure and description of an arbitrary off-chain storage handler tasked with the responsibility of writing to an arbitrary storage. First introduced in EIP-5559, the protocol outlined herein expands the capabilities of the `StorageHandledBy<>()` revert to accept decentralised and namespaced storages. In addition, this draft proposes that besides `StorageHandledByL2()` and `StorageHandledByOffChainDatabase()`, new `StorageHandledBy<>()` reverts be allowed through a publicly curated listed where each new `StorageHandledBy<>()` storage handler must be accompanied by a complete documentation of its interface and design. Some foreseen examples of new storage handlers include `StorageHandledByIPFS()` for IPFS, `StorageHandledByIPNS()` for IPNS, `StorageHandledByArweave()` for Arweave, `StorageHandledByArNS()` for ArNS, `StorageHandledBySwarm()` for Swarm etc.

![](https://raw.githubusercontent.com/namesys-eth/namesys-ccip-write/main/images/schematic.png)

Similar to EIP-5559, a CCIP-Write deferral call to an arbitrary function `setValue(bytes32 key, bytes32 value)` can be described in pseudo-code as follows:

```solidity
// Define revert event
error StorageHandledBy*(...)

// Define metadata API interface
function metadata(
    bytes calldata ... // Reference a user (optional)
)
    external
    view
    returns (...)
{
    // Return on-chain metadata for a user OR, read metadata from off-chain source via
    // CCIP-Read aka 'OffchainLookup()'. If partial metadata exists off-chain, return 
    // may include URL for that data's off-chain API (e.g. ENS off-chain resolvers may
    // rely on GraphQL endpoints to fetch complete off-chain state for a node)
    return (...) | revert OffchainLookup(...);
}

// Generic function in a contract
function setValue(
    bytes32 key,
    bytes32 value
) external {
    // Defer write call to off-chain handler
    revert StorageHandledBy*(...);
}
```

where, the following structure for `StorageHandledBy<>()` must be followed:

```solidity
// Details of revert event
error StorageHandledBy*(
    bytes msg.sender, // Sender of call
    bytes callData, // Payload to store
    bytes4 contract.metadata.selector // Function selector for metadata API
);
```

#### Metadata
The `metadata()` function captures all the relevant information that the client may require to update a user's data on their favourite storage. For instance, `metadata()` must return a pointer to a user's data on their desired storage. In the case of `StorageHandledByL2()` for example, `metadata()` must return a chain identifier such as `ChainId` and additionally the contract address. In case of `StorageHandledByOffChainDatabase()`, `metadata()` must return the custom gateway URL serving a user's data. In case of `StorageHandledByIPNS()`, `metadata()` may return the public key of a user's IPNS container; the case of ArNS is similar. In addition, `metadata()` may further return security-driven information such as a delegated signer's address who is tasked with signing the off-chain data; such signers and their approvals must also be returned for verification tasks to be performed by the client. It follows that each storage handler `StorageHandledBy<>()` must define the precise construction of `metadata()` function in their documentation. Note that the `metadata()` function doesn't necessarily read any or all of the aforementioned metadata from the contract; it is possible that this metadata is in fact stored off-chain, in which case `metadata()` function may instead revert with `OffchainLookup()` that the client must process.

```solidity
// Generic metadata function's construction
function metadata(
    bytes calldata node // Reference a user (optional)
)
    external
    view
    returns (...)
{
    // Return on-chain metadata for a user OR, read entire metadata from off-chain source via
    // CCIP-Read aka 'OffchainLookup()'. If some metadata exists off-chain, return may include
    // URL for that data's off-chain API (e.g. ENS off-chain resolvers implementing wildcard 
    // resolution often rely on GraphQL to fetch complete off-chain state for a node)
    (string metaEndpoint, metadata onchainMetadata) = getMetadata(...);
    return (
        metadata onchainMetadata, // Relevant on-chain metadata (optional)
        string metaEndpoint, // Endpoint URL for metadata API (optional)
        ...
    ) | revert OffchainLookup(...); // If entire metadata exists off-chain
}
```

Some example constructions of `metadata()` functions which support L2, databases, IPFS, Arweave, IPNS, ArNS and Swarm[`?`] are given below.

### L2 Handler: `StorageHandledByL2()`
A mimimal L2 handler only requires the list of `ChainId` values and the corresponding `contract` addresses and `StorageHandledByL2()` as defined in EIP-5559 is sufficient. In context of this proposal, `ChainId` and `contract` must be returned by the `metadata()` function. The deferral in this case will prompt the client to submit the transaction to the relevant L2 as returned by the `metadata()` function. One example of an L2 handler's `metadata()` function is given below.

#### EXAMPLE
```solidity
error StorageHandledByL2(contract.metadata.selector, ...)

function metadata()
    external
    view
    returns (
        address, // Contract address on L2
        string memory, // ChainID of L2
        ... // Metadata API endpoint etc (optional)
    )
{   
    (address contractL2, string chainId) = getMetadata(...);
    // contractL2 = "0x32f94e75cde5fa48b6469323742e6004d701409b"
    // chainId = "21"
    return (
        contractL2, // Contract address on L2
        chainId, // L2 ChainId
        ...
    );
}
```

There may however arise a situation where a service first stores some data on L2 and then writes - asynchronously or otherwise - to another off-chain storage type. In such cases, the L2 contract should implement a second off-chain write deferral after making desired local state changes. This in principle allows creation of chained storage handlers without explicitly introducing a callback function in this proposal.

### Database Handler: `StorageHandledByDatabase()`
A minimal database handler is similar to an L2 in the sense that:

  a) it requires the gateway `url` responsible for handling off-chain write operations (similar to `ChainId`), and

  b) it should require `eth_sign` output to secure the data and the client must prompt the users for these signatures (similar to `eth_call`).

In this case, the `metadata()` must return the bespoke `url` and may additionally return the addresses of `signer` of `eth_sign`. If a `signer` is returned by the metadata, then the client must make sure that the signature forwarded to the gateway is signed by that `signer`. One example of a database handler's `metadata()` function is given below.

#### EXAMPLE
```solidity
error StorageHandledByDatabase(contract.metadata.selector, ...)

function metadata()
    external
    view
    returns (
        string memory, // Gateway URL
        address, // Ethereum signer's address
        ... // Metadata API endpoint etc (optional)
    )
{   
    (string gatewayUrl, address signer) = getMetadata(...);
    // gatewayUrl = "https://api.namesys.xyz"
    // signer = "0xc0ffee254729296a45a3885639AC7E10F9d54979"
    return (
        gatewayUrl, // Gateway URL
        signer, // Signer's address
        ...
    );
}
```

In the above example, the client must first verify that the `eth_sign` is signed by a legitimate address in `signer`, then prompt the user for a signature and finally pass the resulting signature to the respective gateway URL. The message payload for the signature in this case may be formatted as per EIP-712, as detailed in EIP-5559. Some storage handlers may however choose simple string formatting as long as it is properly documented in their documentation. This proposal leaves this aspect of off-chain metadata construction to storage handlers and individual ecosystems.

### Decentralised Storage Handlers
Decentralised storages are the extremest in the sense that they come both in immutable and mutable form; the **immutable** forms locate the data through immutable content identifiers (CIDs) while **mutable** forms utilise some sort of namespace which can statically reference any dynamic content. Examples of the former include raw content hosted on IPFS and Arweave while the latter forms use IPNS and ArNS namespaces respectively to reference the raw and dynamic content. 

The case of immutable forms is similar to a database although these forms are not as useful in practise so far. This is due to the difficulty associated with posting the unique CID on chain each time a storage update is made. One way to bypass this difficulty is by storing the CID cheaply in an L2 contract; this method requires the client to update the data on both the decentralised storage as well as the L2 contract through two chained deferrals. CCIP-Read in this case is also expected to read from two storages to be able to fully handle a read call. Contrary to this tedious flow, namespaces can instead be used to statically fetch immutable CIDs. For example, instead of a direct reference to immutable CIDs, IPNS and ArNS public keys can instead be used to refer to IPFS and Arweave content respectively; this method doesn't require dual deferrals by CCIP-Write (or CCIP-Read), and the IPNS or Arweave public key needs to be stored on chain only once. However, accessing the IPNS and ArNS content now requires that the client must prompt the user for additional information via `context`, e.g. IPNS and ArNS signatures in order to update the data.

Decentralised storage handlers' `metadata()` interface is therefore expected to return additional `context` which the clients must interpret and evaluate before calling the gateway with the results. This feature is not supported by EIP-5559 and services using EIP-5559 are thus incapable of storing data on decentralised namespaced & mutable storages. One example of a decentralised storage handler's `metadata()` function for IPNS is given below.

#### EXAMPLE (`StorageHandledByIPNS()`)
```solidity
error StorageHandledByIPNS(contract.metadata.selector, ...)

function metadata()
    external
    view
    returns (
        string memory, // Gateway URL
        address, // Signer of off-chain data
        bytes memory, // Context for namespace
        ... // Metadata API endpoint etc (optional)
    )
{   
    (string gatewayUrl, address ethSigner, bytes ipnsSigner) = getMetadata(...);
    // gatewayUrl = "https://ipns.namesys.xyz"
    // ethSigner = "0xc0ffee254729296a45a3885639AC7E10F9d54979"
    // ipnsSigner = "0xe50101720024080112203fd7e338b2de90159832ffcc434927da8bbfc3a000fa58ea0548aa8e08f7e10a"
    return (
        gatewayUrl, // Gateway URL
        ethSigner, // Ethereum signer's address
        ipnsSigner, // IPNS signer's hex-encoded CID
        ...
    );
}
```

In this example, the client must process the `context` according to the specifications of the native `StorageHandledBy<>()` identifier. For instance, in the particular example shown above, the client must request the user for at least a `sequence` counter and an IPNS signature matching the signer's CID returned in `context`. The clients should evaluate the `context` by feeding the `sequence` counter to the message payload and then obtaining the resulting IPNS signature. This signature must then be passed to the gateway among other arguments. 

### New Revert Events
1. Each new storage handler must submit their `StorageHandledBy<>()` identifier through an ERC track proposal referencing the current draft and EIP-5559.

2. Each `StorageHandledBy<>()` provider must be supported with detailed documentation of its structure and the necessary `metadata()` that its implementers must return.

3. Each `StorageHandledBy<>()` proposal must define the precise formatting of any message payloads that require signatures and complete descriptions of custom cryptographic techniques implemented for additional security, accessibility or privacy.

### Signatures
`TBA`

### Interfaces
`TBA`

## Backwards Compatibility
`TBA`

## Security Considerations
`TBA`

## Copyright
Copyright and related rights waived via [CC0](../LICENSE.md).