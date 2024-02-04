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

EIP-5559 was the first step toward such a tolerant 'CCIP-Write' protocol which outlined how write deferrals could be made to L2 and centralised databases. The cases of L2 and database are similar; deferral to an L2 involves routing the `eth_call` to L2, while deferral to a database can be made by extracting `eth_sign` from `eth_call` and posting the resulting signature along with the data for later verification. In both cases, no pre-flight information needs to be processed by the client and arguments of `eth_call` and `eth_sign` as specified in EIP-5559 are sufficient. This proposal extends the previous attempt by including secure write deferrals to decentralised storages, especially those which - beyond the arguments of `eth_call` and `eth_sign` - require additional pre-flight metadata from clients to successfully host users' data on their favourite storage. This document also enables more complex and generic use-cases of databases such as those which do not store the signers' addressess on-chain as presumed in EIP-5559.

### Curious Case of Decentralised Storages
Decentralised storages powered by cryptographic protocols are unique in their diversity of architectures compared to centralised databases or L2 chains, both of which have canonical architectures in place. For instance, write calls to L2 chains can be generalised through the use of `ChainID` for any given `callData`; write deferral in this case is as simple as routing the `eth_call` to another contract on an L2 chain. There is no need to incorporate any additional security requirement(s) since the L2 chain ensures data integrity locally, while the global integrity can be proven by employing a state verifier scheme (e.g. EVM-Gateway) during CCIP-Read calls. Centralised databases have a very similar architecture where instead of invoking `eth_call`, the result of `eth_sign` needs to be posted on the database along with the `callData` for integrity verification by CCIP-Read.

Decentralised storages on the other hand, do not typically have EVM- or database-like environments and may have their own unique content addressing requirements. For example, IPFS, Arweave, Swarm etc all have unique content identification schemes as well as their own specific fine-tunings and/or choices of cryptographic primitives, besides supporting their own cryptographically secured namespaces. This significant and diverse deviation from EVM-like architecture results in an equally diverse set of requirements during both the write deferral operation as well as the subsequent state verifying stage.

For example, consider a scenario where the choice of storage is IPNS or ArNS. In precise terms, IPNS storage refers to immutable IPFS content wrapped in mutable IPNS namespace, which eventually serves as the reference coordinate for off-chain data. The case of ArNS is similar; ArNS is immutable Arweave content wrapped in mutable ArNS namespace. To write to IPNS or ArNS storage, the client requires more information than only the gateway URL `url` responsible for write operations and arguments of `eth_sign`. More precisely, the client must at least prompt the user for their IPNS or ArNS signature which is necessary for updating the namespaced storage. The client may also require additional information from the user such as specific arguments required by IPNS or ArNS signature. One such example is the requirement of integer `sequence` of IPNS update which goes into the construction of signature message payload. These additional user-centric requirements are not accommodated by EIP-5559, and the resolution of these issues -- among others such as batch writing -- is detailed in the following attempt towards a suitable CCIP-Write specification.

## Specification    
### Overview
The following specification revolves around the structure and description of an arbitrary off-chain storage handler tasked with the responsibility of writing to an arbitrary storage. First introduced in EIP-5559, the protocol outlined herein expands the capabilities of the `StorageHandledBy__()` revert to accept decentralised and namespaced storages. In addition, this draft proposes that besides `StorageHandledByL2()` and `StorageHandledByOffChainDatabase()`, new `StorageHandledBy__()` reverts be allowed through a publicly curated listed (similar to multiformats and multicodec library) where each new `StorageHandledBy__()` storage handler must be accompanied by a complete documentation of its interface and design. Some foreseen examples of new storage handlers include `StorageHandledByIPFS()` for IPFS, `StorageHandledByIPNS()` for IPNS, `StorageHandledByArweave()` for Arweave, `StorageHandledByArNS()` for ArNS, `StorageHandledBySwarm()` for Swarm etc.

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
    config onChainConfig = getInfoFromContract(...)
    // Defer write call with relevant on-chain information
    revert StorageHandledBy__(onChainConfig, ...)
}
```

where, the following structure for `StorageHandledBy__()` must be followed:

```solidity
// Details of Revert Event
error StorageHandledBy__(
    bytes msg.sender, // Sender of call
    bytes callData, // Payload to store
    config onChainConfig // Send all necessary data from contract
)
```

The type `config` captures all the relevant information that the client may require from the contract to update a user's data on their favourite storage. For instance, `config` should contain the public coordinates to the user's data if such data exists on-chain. In case of `StorageHandledByIPNS()` for example, `config` may contain the public key of a user's IPNS container that has been stored on-chain; the case of ArNS is similar. In case of `StorageHandledByOffChainDatabase()`, `config` may contain the custom gateway URL serving a user's data or some form of unique identifier required by the client to locate the user's data. It follows that each storage handler `StorageHandledBy__()` must define the precise construction of their chosen `config` in their documentation.

### L2 Handler
A mimimal L2 handler only requires the list of `ChainID` values and the corresponding `contract` addresses and `StorageHandledByL2()` as defined in EIP-5559 is sufficient. In context of this proposal, `ChainID` and `contract` must be part of the `config`. There may however arise a situation where a service first stores some data on L2 and then writes - asynchronously or otherwise - to another off-chain storage type; in such cases, `config` may additionally contain the necessary metadata to write to off-chain storage.



### Database Handler
In the minimal version, a database handler only requires the list of URLs (`urls`) responsible for the write operations. However, it is strongly advised that all clients employ some sort of verifiable signature scheme and sign the off-chain data; these signatures can be verified during CCIP-Read calls and will prevent possible unauthorised alterations to the data. In such a scenario, the list of signatures (`approvals`) are needed at the very least; if the signing authority is **not** stored on-chain, then the address of the authority (`authorities`) must also be attached.

```solidity
revert StorageHandledByDB(
    address msg.sender,
    [
        string[], 
        address[], 
        bytes[], 
        bytes[]
    ] [
        urls, // List of URLs handling writing to databases
        signers || [], // List of addresses signing the calldata
        approvals || [], // List of signatures approving the signers
        [] // MUST be empty for centralised databases
    ],
    bytes callData,
    bytes4 this.callback.selector,
    bytes extraData
)

function callback(...) external view {
    ...
    return
}
```

#### EXAMPLE
```solidity
function setValueWithConfig(
    bytes32 key, 
    bytes32 value,
    [
        string[], 
        address[], 
        bytes[], 
        bytes[]
    ] [
        urls,
        signers,
        approvals,
        []
    ],
) external {
    revert StorageHandledByDB(
        msg.sender,
        abi.encodePacked(value),
        [
            urls, 
            signers,
            approvals,
            []
        ],
        this.callback.selector,
        extraData
    )
}

function callback(
    bytes response,
    config config,
    bytes extraData
) external view {
    bytes newConfig = calculateOutputForDB(response, config, extraData)
    return (
        newConfig,
        response == true
    )
}
```

#### CALL a DB
```solidity
setValueWithConfig(
    "avatar", 
    "https://namesys.xyz/logo.png",
    [
        string[], 
        address[], 
        bytes[], 
        bytes[]
    ] [
        [
            "https://db.namesys.xyz", // Database 1   
            "https://db.notapi.dev" // Database 2
        ], 
        [
            "0xc0ffee254729296a45a3885639AC7E10F9d54979", // Ethereum Signer 1
            "0x75b6B7CEE3719850d344f65b24Db4B7433Ca6ee4" // Ethereum Signer 2
        ],
        [
            "0xa6f5e0d78f51c6a80db0ade26cd8bb490e59fc4f24e38845a6d7718246f139d8712be7a3421004a3b12def473d5b9b0d83a0899fb736200a915a1648229cf5e21b", // Approval Signature 1
            "0x8d591768f97f950d1c2cb8a51e4f8718cd154d07e0b60ec955202ac478c45b6f3b745ee136276cbfc4a7c1d7c7cdd0a8e8921395b60c556f0c4857ead0447e351c" // Approval Signature 2
        ],
        [] // MUST be empty for centralised databases
    ]
)
```

### Decentralised Storage Handler
Decentralised storage handlers are the most advanced case and require an equivalent config to database handlers. In addition, such storages are usually immutable at core (e.g. IPFS and Arweave) and therefore employ cryptographic namespaces for static data retrieval. Such namespaces typically have their own access keypairs and their own choices of base-encodings as well as elliptic curves. In order to write to such storages wrapped in namespaces, signature and other relevant metadata (`accessories`) must be included in the config.

```solidity
revert StorageHandledByXY(
    address msg.sender,
    [
        string[], 
        address[], 
        bytes[], 
        bytes[]
    ] [
        urls, // List of URLs handling write operations to off-chain storages
        signers || [], // List of addresses signing the calldata
        approvals || [], // List of signatures approving the signers
        accessories || [] // List of access signatures for native namespaces
    ],
    bytes callData,
    bytes4 this.callback.selector,
    bytes extraData
)

function callback(...) external view {
    ...
    return
}
```

#### EXAMPLE
```solidity
function setValueWithConfig(
    bytes32 key, 
    bytes32 value,
    [
        string[], 
        address[], 
        bytes[], 
        bytes[]
    ] [
        urls,
        signers,
        approvals,
        accessories
    ],
) external {
    revert StorageHandledByXY(
        msg.sender,
        abi.encodePacked(value),
        [
            urls, 
            signers,
            approvals,
            accessories
        ],
        this.callback.selector,
        extraData
    )
}

function callback(
    bytes response,
    config config,
    bytes extraData
) external view {
    bytes newConfig = calculateOutputForXY(response, config, extraData)
    return (
        newConfig,
        response == true
    )
}
```

#### CALL IPNS and ArNS
```solidity
setValueWithConfig(
    "avatar", 
    "https://namesys.xyz/logo.png",
    [
        string[], 
        address[], 
        bytes[], 
        bytes[]
    ] [
        [
            "https://ipns.namesys.xyz", // IPFS-NS Write Gateway    
            "https://arweave.notapi.dev" // Arweave-NS Write Gateway
        ], 
        [
            "0xc0ffee254729296a45a3885639AC7E10F9d54979", // Ethereum Signer for IPFS
            "0x1CFe432f336cdCAA3836f75A303459E61077068C" // Ethereum Signer for Arweave
        ],
        [
            "0xa6f5e0d78f51c6a80db0ade26cd8bb490e59fc4f24e38845a6d7718246f139d8712be7a3421004a3b12def473d5b9b0d83a0899fb736200a915a1648229cf5e21b", // Approval Signature for IPFS
            "0xf4e42fa7d1125fc149f29ed437e8cbbdac7e31bb493299e03df4d8cfd069c9a96bb12f6186e79ed6bc6e740086a67c0da022ffcd84ef50abf6c0e4f83d53a62d1c" // Approval Signature for Arweave
        ],
        [
            abi.encodePacked(
                "0xa74f6d477c01189834a56b52c8189d6fb228d40e17ef0b255b36848f1432f0bc35b1cf4a2f5390a8aef6c72665b752907be6a979a3ff180d9c13c7983df5d9c2", // Hex-encoded IPNS signature over ed25519 curve
                bytes32(1) // Index or sequence or version number required by IPNS signature payloads; give bytes(0) for empty value
            ), // Requires casting to bytes-like payload by gateway for IPFS-NS
            abi.encodePacked(
                "0x8a055b79515356324f68c18071b22085607d4f37577d53fe5c5c2b0ec9769ef1e70a5bc53f9fe901051e493a216a02ae7952a62488e26fa9547e504af01ef25cd904d853ea409fdf23bec0929caae4926d5e8e5353b4663880a", // Hex-encoded Arweave signature over ed25519 curve
                bytes32(0) // Not required for Arweave
            ) // Requires casting back to base64 by gateway for Arweave
        ]
    ]
)
```

### Events
1. A public library must be maintained where each new storage handler supported by a native Protocol Improvement Proposal must register their `StorageHandledBy__()` identifier. This library could exist on-chain or off-chain; in the end such a list of `StorageHandledBy__()` identifiers must be the accepted standard for CCIP-Write infrastructure providers (similar to multiformats & multicodec table). If the 2-character space runs out, it can be extended to 3 or more characters without any fear of identifier collisions.

2. Each `StorageHandledBy__()` provider should be supported with detailed docs of their infrastructure along with a Protocol Improvement Proposal.

### Interface
`TBA`

## Backwards Compatibility
`TBA`

## Security Considerations
`TBA`

## Copyright
Copyright and related rights waived via [CC0](../LICENSE.md).