---
DESCRIPTION: Secure off-chain storage handler capable of writing to arbitary non-EVM storage
---

# ENSIP-Y: Off-Chain Data Write Handler for Mutable Storages

| **Author**    | `@sshmatrix <1@2.3>`, `@0xc0de4c0ffee <a@b.c>` |
| ------------- | ---------------------------------------------------------------- |
| **Status**    | Draft                                                            |
| **Submitted** | ◥                                                                |


## Abstract
This proposal outlines the methods that clients should implement in order to handle `StorageHandledByDatabase()` and `StorageHandledByIPNS()` reverts made by off-chain resolvers. Methods in this document are designed to securely and autonomously incorporate off-chain storages - in particular databases and decentralised storages - in ENS resolvers. By implementing these methods in their resolvers to store records, ENS service providers can take advantage of cheap and often free nature of most off-chain storages.

## Motivation
Gas fees are a burden on ENS users and off-chain resolvers such as CB.ID and NameSys have recently attempted to tackle this problem. CB.ID for instance offers off-chain records for theirs users on L2 and their centralised databases following EIP-5559. However, EIP-5559 is incapable of defering storage to decentralised storages such as those used by NameSys, e.g. IPFS, Arweave or Swarm. In order to support decentralised storages in particular, EIP-5559 falls short both in terms of UX as well as protocol interface. EIP-X updates EIP-5559 by introducing storage handlers for L2, databases and decentralised storages through which L1 contracts can store data off chain at significantly lower cost. This approach when combined with EIP-3668 (CCIP-Read) has the potential to nullify the gas associated with storing ENS records without compromising security and autonomy. The following text describes how ENS off-chain resolvers should implement `StorageHandledByDatabase()` and `StorageHandledByIPNS()` handlers as defined in EIP-X to store users' records. The architecture described in this proposal forms the backbone of NameSys `v2` Resolver.

## Specification
### Overview
`StorageHandledByIPNS()` is more complex in construction than `StorageHandledByDatabase()` which is a reduced version of the former. For this reason, we still start by describing how ENS clients should implement `StorageHandledByIPNS()` first. Later on, we will reduce the requirements to the simpler case of `StorageHandledByDatabase()`.

### Revert `StorageHandledByIPNS()`
Consider the following example of an ENS resolver implementing `StorageHandledByIPNS()` to store records on IPNS.

```solidity
interface iResolver {
    // Defined in EIP-X
    error StorageHandledByIPNS(
        address sender,
        bytes callData,
        bytes4 contract.metadata.selector
    );
    // Defined in EIP-137
    function setAddr(bytes32 node, address addr) external;
}

// Defined in EIP-X
string public gatewayUrl = "https://api.namesys.xyz";
string public graphqlApi = "https://gql.namesys.xyz";

/** 
* Metadata interface required by off-chain clients as defined in EIP-X & ENSIP-16
* @param node Namehash of ENS domain to fetch metadata for
* @return metadata Metadata required by off-chain clients. Clients must refer to
* ENSIP-Y for directions to process the returned metadata
*/
function metadata(bytes calldata node)
    external
    view
    returns (string memory, address, bytes memory, string memory)
{   
    // Get ethereum signer & IPNS CID stored on-chain with arbitrary logic/code
    address dataSigner = metadata[node].dataSigner; // Unique to each name
    bytes ipnsSigner = metadata[node].ipnsSigner; // Unique to each name or each owner address
    return (
        gatewayUrl, // Gateway URL tasked with writing to IPNS
        dataSigner, // Ethereum signer's address
        ipnsSigner, // IPNS signer's hex-encoded CID as context for namespace
        graphqlApi // GraphQL metadata endpoint (required by ENSIP-16)
    );
}

/**
* Sets the ethereum address associated with an ENS node
* [!] May only be called by the owner or manager of that node in ENS registry
* @param node Namehash of ENS domain to update
* @param addr Ethereum address to set
*/
function setAddr(
    bytes32 node,
    address addr
) authorised(node) {
    // Defer to IPNS storage
    revert StorageHandledByIPNS(
        msg.sender,
        abi.encode(node, addr),
        iResolver.metadata.selector
    );
}
```

In this example, when a user calls `setAddr()` with some values for `node` and `addr`, the resolver reverts with `StorageHandledByIPNS()` containing the original data in encoded form (`abi.encode(node, addr)`), sender of the call (`msg.sender`) and a pointer to fetching the metadata corresponding to the provided `node`. The client upon receiving this revert must decode the calldata and fetch the metadata by interpreting the bespoke pointer which is a `bytes4` indentifier for the function responsible for providing the metadata to the client.

The `metadata()` function tasked with returning the metadata to the clients spits out at most four parameters:

1. `gatewayUrl`: URL of gateway responsible for performing write operations to IPFS network,

2. `dataSigner`: address of signer of off-chain data; this address must be verified by CCIP-Read-enabled resolvers (EIP-3668) using `ecrecover()`,

3. `ipnsSigner`: hex-encoded IPNS CID (ENSIP-7) derived from the `ed25519` public key associated with IPNS namespace, and

4. `graphqlApi`: string-formatted URL of GraphQL API endpoint broadcasting all other metadata required by ENSIP-16, e.g. information about subdomains for wildcard resolvers (ENSIP-10), `sequence` counter of IPNS update required by EIP-X etc.

By the end of this step, the client should have all the necessary parameters for interpreting the metadata. The methods in the following section describe the steps for said interpretation.

### Interpreting Metadata
The methods described in this section have been designed with autonomy, privacy, UI/UX and accessibility for end ENS users in mind. The plethora of off-chain storages have their own diverse ecosystems such that it in not uncommon for each storage to have its own set of UI/UX requirements, such as wallets, signer extensions etc. If users of ENS were to utilise such storage providers, they will inevitably be subjected to additional wallet extensions in their browsers. This is not ideal and the methods in this section have been crafted such that users do not need to install any additional UI/UX components or extensions other than their favourite ethereum wallet.

This draft proposes that both the `dataSigner` and `ipnsSigner` be generated deterministically from ethereum wallet signatures. 

![](https://raw.githubusercontent.com/namesys-eth/namesys-ccip-write/main/images/keygen.png)

## Backwards Compatibility
`TBA`

## Security Considerations
`TBA`

## Copyright
Copyright and related rights waived via [CC0](../LICENSE.md).