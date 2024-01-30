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
Decentralised storages powered by cryptographic protocols are unique in their diversity of architectures compared to centralised databases or L2 chains, both of which have canonical architectures in place. For instance, write calls to L2 chains can be generalised through the use of `ChainID` since the `callData` remains the same; write deferral in this case is as simple as routing the call to another contract on an L2 chain. There is no need to incorporate any additional security requirement(s) since the L2 chain ensures data integrity locally, while the global integrity can be proven by employing a state verifier scheme (e.g. EVM-Gateway) during CCIP-Read calls. 

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
function setValueWithConfig(
    bytes32 key, 
    bytes32 value,
    config config // Metadata pertaining to storage handler __
) external {
    // Defer another write call to storage handler
    revert StorageHandledBy__(
        this.callback.selector, 
        config config,
        ...
    )
}

// Callback receiving status of write call
function callback(bytes response, ...) external view {
    return
} 
```

### Interdependence
The condition of interdependence on storage handlers requires that each handler must have a global `config` interface in the input argument as well as the return statement. This requires that `StorageHandledBy__()` must be of the form

- `error StorageHandledBy__(bytes config, ...)`, in addition to 
- `function callback(bytes config, ...) returns (bytes memory newConfig, ...)`, 

where `config` and `newConfig` are responsible for the interdependent behaviour. We'll specify the optimal typing and encoding for these two interfaces in the next section, although both payloads must have the exact same encoding and may include not only data but also metadata governing the behaviour of subsequent asynchronous calls to nested handlers.

![](https://raw.githubusercontent.com/namesys-eth/namesys-ccip-write/main/images/nested.png)

In pseudo-code, interdependent and nested CCIP-Write deferral looks like:

```solidity
// Define Revert Events for storages X1 and X2
error StorageHandledByX1(
    address sender, 
    config config, // Metadata pertaining to storage handler X1
    bytes callData, 
    bytes4 callback, 
    bytes extraData
)
error StorageHandledByX2( 
    address sender, 
    config config, // Metadata pertaining to storage handler X2
    bytes callData, 
    bytes4 callback, 
    bytes extraData
)

// Generic function in a contract
function setValue(
    bytes32 key, 
    bytes32 value,
    config config
) external {
    // Defer write call to X1 handler
    revert StorageHandledByX1(
        address(this),
        config config,
        abi.encodePacked(value),
        this.callback.selector,
        extraData,
        ...
    )
}

// Callback receiving response from X1
function callback(
    bytes response, 
    config config, 
    bytes extraData
) external view {
    (config config2, bytes puke, bytes extraData2) = calculateOutputForX1(response, config, extraData)
    // Defer another write call to X2 handler
    revert StorageHandledByX2(
        address(this),
        config config2,
        abi.encode(puke),
        this.callback2.selector,
        extraData2,
        ...
    ) || return (config2, puke, ...)
} 

// Callback receiving response from X2
function callback2(
    bytes response2, 
    config config2, 
    bytes extraData2
) external view {
    (config config3, bytes puke2, bytes extraData2) = calculateOutputForX2(response2, config2, extraData2)
    // Defer another write call to X3 handler
    revert StorageHandledByX3(
        address(this),
        config config3,
        abi.encode(puke2),
        this.callback3.selector,
        extraData3,
        ...
    ) || return (config3, puke2, ...)
} 

// Callback receiving response from X3
function callback3(...) external view {
    ...
    return
}
```

### Config Interface
Config interface is dedicated to handling all the metadata that different storages require to defer the calls successfully. Keeping in mind the needs of a broad set of storages, `config` interface consists of four arrays:

1. `coordinates`: Coordinates refer to the string-formatted pointers to the target storage. For example, for writing to an L2, its `ChainID` is sufficient information. For writing to a database or a decentralised storage, the handler's HTTP URL is sufficient information. 

2. `authorities:` Authorities refer to the addresses of authorities securing the off-chain data. For an L2, the contract address is the authority. For a database or decentralised storage, the data should ideally be signed by an Ethereum private key, which is also the authority in this case. If the authority is stored on-chain by some contracts, then this value is not needed. 

3. `approvals:` Approvals refer to the signatures signed by the corresponding authorities, irrespective of whether they are stored on-chain or not. 

4. `accessories:` Accessories refer to the metadata required to update the off-chain storages, if the storage is wrapped in a namespace. For example, for IPFS wrapped in IPNS, signature from the IPNS private key along with the last sequence number is the required accessory. For databases, login credentials are the accessories although clients must be careful not to pass raw login credentials and use encryption strategies. 

```solidity
// Type of config
type config = [
        string[], 
        address[], 
        bytes[], 
        bytes[]
    ] 
// Data inside config
bytes[] config = [
        coordinates, // List of string-formatted coordinates, e.g. ChainID for L2, URL for off-chain storage etc; can never be empty
        authorities || [], // List of addresses of authorities; can be empty for unsafe record storage in off-chain databases or decentralised storages, or for on-chain signers
        approvals || [], // List of bytes-like signatures/approvals; can be empty for unsafe record storage in off-chain databases or decentralised storages
        accessories || [] // List of bytes-like access signatures for accessories; usually empty except for decentralised storages wrapped in namespaces
    ]
```

### L2 Handler
L2 handler only requires the list of `ChainID` values and the corresponding `contract` addresses.

```solidity
revert StorageHandledByL2(
    address sender,
    [
        string[], 
        address[], 
        bytes[], 
        bytes[]
    ] [
        chains, // List of string-formatted ChainID values
        contracts, // List of contracts on L2
        [], // MUST be empty for L2 storage handler
        [] // MUST be empty for L2 storage handler
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
        chains, 
        contracts,
        [],
        []
    ],
) external {
    revert StorageHandledByL2(
        msg.sender,
        [
            chains,
            contracts,
            [],
            []
        ],
        abi.encodePacked(value),
        this.callback.selector,
        extraData
    )
}

function callback(
    bytes response,
    config config,
    bytes extraData
) external view {
    bytes newConfig = calculateOutputForL2(response, config, extraData)
    return (
        newConfig,
        response == true
    )
}
```

#### CALL an L2
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
            "11", // ChainID for chain 1
            "25" // ChainID for chain 2
        ], 
        [
            "0xc0ffee254729296a45a3885639AC7E10F9d54979", // Contract address on chain 1
            "0x999999cf1046e68e36E1aA2E0E07105eDDD1f08E" // Contract address on chain 2
        ],
        [], // MUST be empty for L2
        [] // MUST be empty for L2
    ]
)
```

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

### Nested Handlers
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
    // 1st deferral
    revert StorageHandledByDB(
        msg.sender,
        abi.encodePacked(value),
        [
            urls, 
            signers,
            approvals,
            []
        ],
        this.callbackDB.selector,
        extraData
    )
}

// Get response after 1st deferral and post-process
function callbackDB(
    bytes response,
    config config,
    bytes extraData
) external view {
    // Calculate output and access signatures for XY's namespaces
    (
        bytes newExtraData, 
        bytes accessories, 
        config config
    ) = calculateOutputForDB(response, config, extraData)
    // 2nd deferral
    revert StorageHandledByXY(
        msg.sender,
        abi.encodePacked(value),
        [
            config.urls, 
            config.signers,
            config.approvals,
            config.accessories
        ],
        this.callbackXY.selector,
        newExtraData
    )
}

// Get response after 2nd deferral and post-process
function callbackXY(
    bytes response,
    config config,
    bytes extraData
) external view {
    // Calculate final output
    bytes newConfig = calculateOutputForXY(response, config, extraData)
    // Final return
    return (
        newConfig,
        response == true
    )
}
```

### NOTES
#### CALL L2 nested in IPNS
In this explicit example, let's consider a scenario where a service intends to publish the `value` of a `key` off-chain with `setValueWithConfig()` to an IPNS container. IPNS updates are notoriously frivolous in their uptake rate at long TTLs and can return stale results for a while in some cases after an update. Due to this reason, the service wants to index at least the `sequence` number of updates on an L2 contract and perhaps also the IPFS hash, from where CCIP-Read can force-match the expected latest version against the resolved IPNS records before resolving any request. This document envisions the process entailed as follows:

1. User makes a request to set `value` of a `key` with `setValueWithConfig()`, and attaches the `config` with legitimate `coordinates` of gateways in form of `urls`, off-chain `signers` as `authorities`, `approvals` for off-chain signers, and `accessories` required to update their IPNS container (IPNS signature + `sequence` number).

    ```solidity
    config config = [
            ["https://ipns.namesys.xyz"], // Gateway URL
            ["0xc0ffee254729296a45a3885639AC7E10F9d54979"], // Signer address
            ["0xa6f5e0d78f51c6a80db0ade26cd8bb490e59fc4f24e38845a6d7718246f139d8712be7a3421004a3b12def473d5b9b0d83a0899fb736200a915a1648229cf5e21b"], // Ethereum signature by signer
            [
                abi.encodePacked(
                    "0xa74f6d477c01189834a56b52c8189d6fb228d40e17ef0b255b36848f1432f0bc35b1cf4a2f5390a8aef6c72665b752907be6a979a3ff180d9c13c7983df5d9c2", // Hex-encoded IPNS signature over ed25519 curve
                    bytes32(1) // Sequence required by IPNS signature payloads
                )
            ]
        ]
    ```

2. `setValueWithConfig()` defers the storage to `StorageHandledByIP()`.

    ```solidity
    function setValueWithConfig(
        bytes32 key, 
        bytes32 value,
        config config
    ) external {
        // 1st deferral
        revert StorageHandledByIP(
            msg.sender,
            abi.encodePacked(key, value),
            config config,
            this.callbackIP.selector,
            extraData
        )
    }
    ```

3. `callbackIP()` receives the `response` from first deferral which contains the new `sequence` of the update, along with updated `newConfig` and `extraData`. Since the next step is L2, `newConfig` containing the relevant information must be returned by the gateway; in this case the relevant information is `ChainID` and `contract` address of the target L2. With `newConfig` in place, `callbackIP()` makes 2nd deferral to L2.

    ```solidity
    // Get response after 1st deferral and post-process
    function callbackIP(
        bytes response,
        config newConfig,
        bytes extraData
    ) external view {
        // 2nd deferral
        revert StorageHandledByL2(
            msg.sender,
            abi.encodePacked(response), // Sequence number (and IPFS hash) to update on L2
            [
                ["11"], // Expects list of ChainID values
                ["0xc0ffee254729296a45a3885639AC7E10F9d54979"], // Expects list of addresses
                [], // MUST be empty for L2
                [] // MUST be empty for L2
            ],
            this.callbackL2.selector,
            newExtraData || extraData // Calculate newExtraData if necessary
        )
    }
    ```

4. `callbackL2()` receives the response of second deferral and post-processes the results accordingly. For instance, if second L2 deferral has failed for some reasons, `callbackL2()` may choose to undo the first deferral with another (third) deferral. If the second deferral has succeeded, it may choose to emit a custom event or perform other tasks before wrapping up.

    ```solidity
    // Get response after 2nd deferral and post-process accordingly
    function callbackL2(
        bytes response,
        config newConfig,
        bytes extraData
    ) external view {
        // Post-process response, newConfig and extraData if required
        doStuff(response, newConfig, extraData)
        return
    }
    ```

#### Nesting, Complexity & Fidelity
Nesting adds significant complexity to the protocol in the sense that the first gateway itself could have internally carried out the tasks performed by second deferral. This is a valid point and there is no correct answer, and there are several pros and cons to either approach. For instance, 

1. **Fidelity**: Without nesting, one can imagine that future services may combine two or more services A, B & C in different orders and each ordered set then requires a new gateway along its arbitrary construction and standardisation. With nesting, 'atomic' handlers can be standardised once during their introduction to the library and then all future services can play with the resulting fidelity without a need for a new gateway standard each time. 

2. **Transparency**: Standard 'atomic' handlers will generally be more transparent to the end user when compared to complex gateways with several stacked storages A, B, C etc under the hood, unless each gateway is well-documented and standardised.

3. **Complexity**: Nesting is complex and adds burden on the protocol, and may need CCIP-Write service providers to support an array of third-party non-EVM services that some handlers may require.

In the end, it is for the developer community to balance their need for fidelity with covariant complexity.

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