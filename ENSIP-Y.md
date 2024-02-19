---
DESCRIPTION: Helper resources for EIP-5559 applied to ENS
---

# ENSIP-Y: Data Write Handler for Decentralised Mutable Storages

| **Author**    | `@sshmatrix <1@2.3>`, `@0xc0de4c0ffee <a@b.c>` |
| ------------- | ---------------------------------------------------------------- |
| **Status**    | Draft                                                            |
| **Submitted** | ◥                                                                |


## Abstract
This proposal outlines the methods that clients should implement in order to handle `StorageHandledByDatabase()` and `StorageHandledByIPNS()` reverts made by off-chain resolvers. Methods in this document are designed to securely and autonomously incorporate off-chain storages - in particular databases and decentralised storages - in ENS resolvers. By implementing these methods in their resolvers to store records, ENS service providers can take advantage of cheap and often free nature of most off-chain storages.

## Motivation
Gas fees are a burden on ENS users and off-chain resolvers such as CB.ID and NameSys have recently attempted to tackle this problem. CB.ID for instance offers off-chain records for theirs users on L2 and their centralised databases following the legacy EIP-5559. However, legacy EIP-5559 is incapable of defering storage to decentralised storages such as those used by NameSys, e.g. IPFS, Arweave or Swarm. In its support for decentralised storages in particular, EIP-5559 falls short both in terms of UX as well as protocol interface. The new EIP-5559 introduces storage handlers for L1s, L2s, databases and decentralised storages through which L1 contracts can store data off chain at significantly lower cost. This approach when combined with EIP-3668 (CCIP-Read) has the potential to nullify the gas associated with storing ENS records without compromising security and autonomy. The following text describes how ENS off-chain resolvers should implement `StorageHandledByDatabase()` and `StorageHandledByIPNS()` handlers as defined in updated EIP-5559 to store users' records. The architecture described in this proposal forms the backbone of NameSys `v2` Resolver.

## Specification
### Overview
EIP-5559 describes **five** parameters that should be defined by specific protocols: `username`, `protocol`, `dataType`, `metadataUrl` interface and `POST` request formatting. The following text defines these five parameters for ENS.

### Username: `username`
`username` must be auto-filled by the client for ENS implementions of EIP-5559. This public field sets the IPNS namespace for a specific implementation.

```js
// Username is dependent on the storage type which can be 'WalletType' or
// 'NodeType'. See definitions in EIP-5559.
// For ENS: node = namehash(normalise("domain.eth")), aka preimage(node) = "domain.eth"
let username;
if (storage === 'WalletType') username = caip10;
if (storage === 'NodeType') username = "domain.eth";
```

where CAIP-10 identifier `caip10` should be derived from the connected wallet's checksummed address `wallet` and string-formatted `chainId` according to:

```js
// CAIP-10 identifier
const caip10 = `eip155:${chainId}:${wallet}`;
```

### Protocol: `protocol`

### Data Type: `dataType`

### `POST` Request 

### `metadataUrl` Interface

## Backwards Compatibility
`TBA`

## Security Considerations
`TBA`

## Copyright
Copyright and related rights waived via [CC0](../LICENSE.md).