---
eip:
title: "Off-Chain Data Write Protocol"
description: Update to Cross-Chain Write Deferral Protocol (EIP-5559) incorporating secure write deferrals to centralised databases and decentralised (and mutable) storages
author: (@sshmatrix), (@0xc0de4c0ffee), (@arachnid)
discussions-to:
status: Draft
type: Standards Track
category: ERC
created:
requires:
---

## Abstract
The following proposal is a generalised revision to EIP-5559: Off-Chain Write Deferral Protocol, targeting a wider set of storage types and introducing security measures to consider for secure off-chain write deferral and retrieval. While EIP-5559 is limited to referring write operations to L2 EVM chains and centralised databases, methods in this document enable secure write deferral to generic decentralised storages - mutable or immutable - such as IPFS, Arweave, Swarm etc. This draft alongside EIP-3668 is a significant step toward a complete and secure infrastructure for off-chain data retrieval and write deferral.

## Motivation
EIP-3668, or 'CCIP-Read' in short, has been key to retrieving off-chain data for a variety of contracts on Ethereum blockchain, ranging from price feeds for DeFi contracts, to more recently records for ENS users. The latter case is more interesting since it dedicatedly uses off-chain storage to bypass the usually high gas fees associated with on-chain storage; this aspect has a plethora of use cases well beyond ENS records and a potential for significant impact on universal affordability and accessibility of Ethereum. 

Off-chain data retrieval through EIP-3668 is a relatively simpler task since it assumes that all relevant data originating from off-chain storages is translated by CCIP-Read-compliant HTTP gateways; this includes L2 chains, centralised databases or decentralised storages. However, each service leveraging CCIP-Read must handle two main tasks externally:

- Writing this data securely to these storage types on their own, and

- Incorporating reasonable security measures in their CCIP-Read compatible contracts for verifying this data before performing on-chain read or write operations.

Writing to a variety of centralised and decentralised storages is a broader objective compared to CCIP-Read largely due to two reasons:

1. Each storage provider typically has its own architecture that the write operation must comply with, i.e. each have their own specific requirements when it comes to writing data to them, and

2. Each storage must incorporate some form of security measures during write operations so that off-chain data's integrity can be verified by CCIP-Read contracts during data retrieval stage.

EIP-5559 was the first step toward such a tolerant 'CCIP-Write' protocol which outlined how write deferrals could be made to L2 and centralised databases. This proposal extends the previous attempt by including secure write deferrals to decentralised storages, while also updating previous specifications with securer alternatives for writing to centralised databases.

### Curious Case of Decentralised Storages
Decentralised storages powered by cryptographic protocols are unique in their diversity of architectures compared to centralised databases or L2 chains, both of which have canonical architectures in place. For instance, write calls to L2 chains can be generalised through the use of `ChainID` since the `calldata` remains the same; write deferral in this case is as simple as routing the call to another contract on an L2 chain. There is no need to incorporate any additional security requirement(s) since the L2 chain ensures data integrity locally, while the global integrity can be proven by employing a state verifier scheme (e.g. EVM-Gateway) during CCIP-Read calls. 

Decentralised storages on the other hand, do not typically have EVM-like environments and may have their own unique content addressing requirements. For example, IPFS, Arweave, Swarm etc all have unique content identification schemes as well as their own specific fine-tunings and/or choices of cryptographic primitives, besides supporting their own cryptographically secured namespaces. This significant and diverse deviation from EVM-like architecture results in an equally diverse set of requirements during both the write deferral operation as well as the subsequent state verifying stage. The resolution of this precise issue is detailed in the following text in an attempt towards a global CCIP-Write specification.

In contrast to EIP-5559, this proposal allows for multiple storage handlers to be nested asynchronously in arbitrary order allowing for maximal interdependence. This feature of interdependence is necessary for highly optimised protocols which employ a mix of two or more storage types at their core. For instance, a service may choose to index cheaply on an L2 while storing the data off-chain entirely; stack-enabled interdependent handlers can achieve such functionality trivially. 

## Specification    
### Overview
The following specification revolves around the structure and description of an arbitrary off-chain storage handler tasked with the responsibility of writing to an arbitrary storage. Similar to CCIP-Read, CCIP-Write protocol outlined herein comprises of 2 molecular parts: initial reversion with error event `StorageHandledBy__()`, which signals the deferral of write operation to the off-chain handler using events, and `callback()` function, which handles operations following the return of initial write deferral. `__` in `StorageHandledBy__` is reserved for two uppercased characters encoding the type of data handle, for example, `__` = `L2` for L2, `DB` for Database, `IP` for IPFS, `BZ` for Swarm, `AR` for Arweave etc; this 2-character identification scheme can accommodate 1000+ different storage types.

![](https://raw.githubusercontent.com/namesys-eth/namesys-ccip-write/main/images/schematic.png)

Following EIP-5559, a CCIP-Write deferral call to an arbitrary function `setValue(bytes32 key, bytes32 value)` can be described in pseudo-code as follows:

```solidity
// Define Revert Event
error StorageHandledBy__(...)

// Generic function in a contract
function setValue(
    bytes32 key, 
    bytes32 value
) external {
    revert StorageHandledBy__(
        this.callback.selector, 
        ...
    )
}

// Callback recieving status of write call
function callback(bytes response, ...) external view {
    return
} 
```

### Interdependence
The condition of interdependence on storage handlers requires that each handler must have a global argument in input as well as the return statement. This requires that `StorageHandledBy__()` must be of the form

- `error StorageHandledBy__(bytes input, ...)`, in addition to 
- `function callback(bytes input, ...) returns (bytes memory output, ...)`, 

where `input` and `output` are the aforementioned arguments responsible for interdependent behaviour. We'll specify the optimal encoding for input and output bytes at a later stage, although both payloads must have the exact same encoding and may include not only data but also metadata governing the behaviour of subsequent asynchronous calls to nested handlers.

![](https://raw.githubusercontent.com/namesys-eth/namesys-ccip-write/main/images/nested.png)

In pseudo-code, interdependent and nested CCIP-Write deferral looks like:

```solidity
// Define Revert Events for storages X1 and X2
error StorageHandledByX1(
    bytes input, 
    address sender, 
    bytes calldata, 
    bytes4 callback, 
    bytes extradata,
    ...
)
error StorageHandledByX2(
    bytes input, 
    address sender, 
    bytes calldata, 
    bytes4 callback, 
    bytes extradata,
    ...
)

// Generic function in a contract
function setValue(
    bytes32 key, 
    bytes32 value
) external {
    // Defer write call to X1 handler
    revert StorageHandledByX1(
        input,
        address(this),
        abi.encodePacked(value),
        this.callback.selector,
        extradata,
        ...
    )
}

// Callback recieving response from X1
function callback(
    bytes response, 
    bytes input, 
    bytes extradata
) external view {
    (bytes output, bytes puke) = calculateOutput(response, input, extradata)
    // Defer another write call to X2 handler
    revert StorageHandledByX2(
        output,
        address(this),
        abi.encode(puke),
        this.callback2.selector,
        extradata2,
        ...
    ) || return (output, puke, ...)
} 

// Callback recieving response from X2
function callback2(
    bytes response2, 
    bytes input2, 
    bytes extradata2
) external view {
    (bytes output2, bytes puke2) = calculateOutput(response2, input2, extradata2)
    // Defer another write call to X3 handler
    revert StorageHandledByX3(
        output2,
        address(this),
        abi.encode(puke2),
        this.callback3.selector,
        extradata3,
        ...
    ) || return (output2, puke2, ...)
} 

// Callback recieving response from X3
function callback3(...) external view {
    ...
    return
}
```

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