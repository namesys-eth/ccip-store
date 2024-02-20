---
DESCRIPTION: ENS-specific implementation of EIP-5559
---

# ENSIP-19: Off-Chain Data Write Handlers for ENS

| **Author**    | `@sshmatrix`, `@0xc0de4c0ffee` |
| ------------- | ---------------------------------------------------------------- |
| **Status**    | Draft                                                            |
| **Submitted** | ◥                                                                |


## Abstract
This proposal outlines the methods that clients should implement in order to handle `StorageHandledByDatabase()` and `StorageHandledByIPNS()` reverts made by off-chain resolvers. Methods in this document are designed to securely and autonomously incorporate off-chain storages - in particular databases and decentralised storages - in ENS resolvers. By implementing these methods in their resolvers to store records, ENS service providers can take advantage of cheap and often free nature of most off-chain storages.

## Motivation
Gas fees are a burden on ENS users and off-chain resolvers such as CB.ID and NameSys have recently attempted to tackle this problem. CB.ID for instance offers off-chain records for theirs users on L2 and their centralised databases following the legacy EIP-5559. However, legacy EIP-5559 is incapable of defering storage to decentralised storages such as those used by NameSys, e.g. IPFS, Arweave or Swarm. In its support for decentralised storages in particular, EIP-5559 falls short both in terms of UX as well as protocol interface. The new EIP-5559 introduces storage handlers for L1s, L2s, databases and decentralised storages through which L1 contracts can store data off chain at significantly lower cost. This approach when combined with EIP-3668 (CCIP-Read) has the potential to nullify the gas associated with storing ENS records without compromising security and autonomy. The following text describes how ENS off-chain resolvers should implement `StorageHandledByDatabase()` and `StorageHandledByIPNS()` handlers as defined in updated EIP-5559 to store users' records. The architecture described in this proposal forms the backbone of NameSys `v2` Resolver.

## Specification
### Overview
EIP-5559 describes **five** parameters that should be defined by specific protocols: `username`, `protocol`, `dataType` and `POST` request formatting. The following text defines these five parameters for ENS, along with a sixth ENS-specific metadata API.

#### 1. `username`
`username` must be auto-filled by the client for ENS implementions of EIP-5559. This public field sets the IPNS namespace for a specific implementation.

```js
/* Username is dependent on the storage type which can be 'WalletType' or 'NodeType'. See definitions in EIP-5559 */
// For ENS: node = namehash(normalise("domain.eth")), aka preimage(node) = "domain.eth"
let username;
if (storage === 'WalletType') username = caip10;
if (storage === 'NodeType') username = "domain.eth";
```

where CAIP-10 identifier `caip10` should be derived from the connected wallet's checksummed address `wallet` and string-formatted `chainId` according to:

```js
/* CAIP-10 identifier */
const caip10 = `eip155:${chainId}:${wallet}`;
```

#### 2. `protocol`
`protocol` is specific to each ENS Resolver's address (`resolver`) and must be formatted as:
```js
/* Protocol identifier */
const protocol = `ens:${chainId}:${resolver}`;
```

#### 3. `dataType`
Data types for ENS are defined by ENSIP-5, ENSIP-7 and ENSIP-9. These are the usual ENS records. 

#### 4. `POST` REQUEST
##### A. `POST` to IPNS
`POST` request for IPNS storage needs to be handled in a custom manner through the `namesys-client` (or `w3name-client`) client-side libraries. This is due to the enforced secret nature of IPNS private key which limits all IPNS related processing to client-side to protect user autonomy. The pseudo-code for autonomous IPNS storage handling is as follows:

```js
/* POST-ing to IPNS */
import IPNS from provider;

let version = "0xa4646e616d65783e6b3531717a693575717535646738396831337930373738746e7064696e72617076366b6979756a3461696676766f6b79753962326c6c6375377a636a73716576616c756578412f697066732f62616679626569623234616272726c7572786d67656461656b667a327632656174707a6f326c35636276646f617934686e70656e757a6f6a7436626873657175656e6365016876616c69646974797818323032352d30312d33305432303a31303a30382e3239315a";
let revision = IPNS.v0() || IPNS.increment(version);
await IPNS.publish(gatewayUrl, revision, IPNS_PRIVATE_KEY);
```

where `IPNS.publish()` takes care of the IPNS signatures and publishing to IPFS network internally.

##### B. `POST` to DATABASE
`POST` request to a RESTful gateway handling database storage must be formatted as:

```ts
/* Type of POST request */
type POST = {
  ens: string
  chainId: number
  approval: string
  records: {
    contenthash: {
      value: string
      signature: string
      timestamp: number
      data: string
    }
    address: [
      {
        coinType: number
        value: string
        signature: string
        timestamp: number
        data: string
      }
    ]
    text: [
      {
        key: number
        value: string
        signature: string
        timestamp: number
        data: string
      }
    ]
  }
}
```
One example of a complete `POST` request to a database is shown below.

```text
/* POST-ing to database */
{
  "ens": "sub.domain.eth",
  "chainId": 1,
  "approval" : "0x1cc5e5efa312dc292560a26e3dba2584070b02ec203c51440a3e23d49ba56b342a4404d8b0d9dc26a94190691e47652343183bf1c64bf9c5081a2f1d887937f11b",
  "records" : {
    "contenthash": {
      "value" : "ipfs://QmRAQB6YaCyidP37UdDnjFY5vQuiBrcqdyoW1CuDgwxkD4",
      "signature": "0x0679eaedb300308680a0e8c11725e891d1500fb98b65d6d09d538e2655567fdf06b989689a01db312ad6df0752cbcb1756b3405a7163f8b4b7c01e70b1a9c5c31c",
      "timestamp": 1708322868,
      "data": "0x2b45eb2b0000000000000000000000005ee86839080d2593b30604e3eeb78271fdc29ec800000000000000000000000000000000000000000000000000000000000000800000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000000000000000018000000000000000000000000000000000000000000000000000000000000000414abb7b2b9fc395910b4387ff69897ee639fe1cf9b79c31bf2d3743134e77a9b222ec175e563d13d60bc722c8829ce91d9af51bcd949816f95979abef4378d84e1c000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000041cbfb2300d60b8602db32ad4ac57279e7a3632e35bb5966eb686e0ac8ec8e7b4a6e306a13a0adee15fce5a9e2bbf3a016db023b0ab66f04bde62a13343287e3851b00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000008000000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000026e3010170122022380b0884d9e85ef3ff5f71ea7a25874738da71f38b999dc8ffec2f6389a3670000000000000000000000000000000000000000000000000000"
    },
    "address": [
      {
        "coinType": 0,
        "value": "1FfmbHfnpaZjKFvyi1okTjJJusN455paPH",
        "signature": "0x60ecd4979ae2c39399ffc7ad361066d46fc3d20f2b2902c52e01549a1f6912643c21d23d1ad817507413dc8b73b59548840cada57481eb55332c4327a5086a501b",
        "timestamp": 1708322877,
        "data": "0x2b45eb2b0000000000000000000000005ee86839080d2593b30604e3eeb78271fdc29ec800000000000000000000000000000000000000000000000000000000000000800000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000000000000000018000000000000000000000000000000000000000000000000000000000000000419c7c185335898d7ec57cffb842e88116a82f367237815f35e16d5f8b28dc3e7b0f0b40edd9f9fc48f771f921986c45973f4c2a82e8c2ebe1732a9f552f8b033a1c000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000041cbfb2300d60b8602db32ad4ac57279e7a3632e35bb5966eb686e0ac8ec8e7b4a6e306a13a0adee15fce5a9e2bbf3a016db023b0ab66f04bde62a13343287e3851b0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000200000000000000000000000001111000000000000000000000000000000000001"
      },
      {
        "coinType": 60,
        "value": "0x839B3B540A9572448FD1B2335e0EB09Ac1A02885",
        "signature": "0xaad74ddef8c031131b6b83b3bf46749701ed11aeb585b63b72246c8dab4fff4f79ef23aea5f62b227092719f72f7cfe04f3c97bfad0229c19413f5cb491e966c1b",
        "timestamp": 1708322917,
        "data": "0x2b45eb2b0000000000000000000000005ee86839080d2593b30604e3eeb78271fdc29ec800000000000000000000000000000000000000000000000000000000000000800000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000000000000000018000000000000000000000000000000000000000000000000000000000000000419bb4494a9ac6b37d5d979cbb6c43cccbbd8790ebbd8f898d8427e1ebfd8bb8bd29a2fbc2b20b0a53c3fdde9dd8ce3df648112754742156d3a5ac6fd1b80d8bd01b000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000041cbfb2300d60b8602db32ad4ac57279e7a3632e35bb5966eb686e0ac8ec8e7b4a6e306a13a0adee15fce5a9e2bbf3a016db023b0ab66f04bde62a13343287e3851b0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000600000000000000000000000000000000000000000000000000000000000000020000000000000000000000000000000000000000000000000000000000000001c68747470733a2f2f6e616d657379732e78797a2f6c6f676f2e706e6700000000"
      }
    ],
    "text": [
      {
        "key": "avatar",
        "value": "https://domain.com/avatar",
        "signature": "0xbc3c7f1b511de151bffe8df033859295d83d400413996789e706e222055a2353404ce17027760c927af99e0bf621bfb24d3bfc52abb36bcfbe6e20cf43db7c561b",
        "timestamp": 1708329377,
        "data": "0x2b45eb2b0000000000000000000000005ee86839080d2593b30604e3eeb78271fdc29ec80000000000000000000000000000000000000000000000000000000000000080000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000000000000001800000000000000000000000000000000000000000000000000000000000000041dc6ca55c1d1c75eec223a7eb01eb5942a2bdb79708c25ff2827cfc0343f97fb76faefd9fbc40de5103956bbdc841f2cc2d53630cd2836a6b76d8d2c107ccadd21b000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000041cbfb2300d60b8602db32ad4ac57279e7a3632e35bb5966eb686e0ac8ec8e7b4a6e306a13a0adee15fce5a9e2bbf3a016db023b0ab66f04bde62a13343287e3851b000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000060000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000046e6e6e6e00000000000000000000000000000000000000000000000000000000"
      },
      {
        "key": "com.github",
        "value": "namesys-eth",
        "signature": "0xc9c33ff219e90510f79b6c9bb489917ee6e00ab123c55abe1117e71ea0d171356cf316420c71cfcf4bd63a791aaf37388ef1832e582f54a8c2df173917240fff1b",
        "timestamp": 1708322898,
        "data": "0x2b45eb2b0000000000000000000000005ee86839080d2593b30604e3eeb78271fdc29ec80000000000000000000000000000000000000000000000000000000000000080000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000000000000001800000000000000000000000000000000000000000000000000000000000000041bfd0ab74712b98bc472ef0e5bbb031acba077fc98a54cdfcb3f11e64b02d7fe21477ba5ea9d508a0265616d74a8df99b9c8f3c04e6bfd41f2df554fe11e1fe141c000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000041cbfb2300d60b8602db32ad4ac57279e7a3632e35bb5966eb686e0ac8ec8e7b4a6e306a13a0adee15fce5a9e2bbf3a016db023b0ab66f04bde62a13343287e3851b000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000060000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000046b6b6b6b00000000000000000000000000000000000000000000000000000000"
      }
    ]
  }
}
```

#### 5. PATHS
EIP-5559 delegates the task of defining the paths for off-chain record files to individual protocols. The `path` scheme for ENS records is based on the RFC-8615 `.well-known` standard. The records for each ENS `sub.domain.eth` must then be stored in JSON format under a reverse-DNS type directory path using `/` instead of `.` as separator. For example, the paths for some example records are formatted as 

- `text/avatar`: `.well-known/eth/domain/sub/text/avatar.json`,
- `contenthash`: `.well-known/eth/domain/sub/contenthash.json`, and
- `address/112`: `.well-known/eth/domain/sub/address/112.json` etc.

#### 6. `metadataUrl` INTERFACE
`metadataUrl` for ENS must point to a GraphQL endpoint and must be formatted as described in ENSIP-16. This `metadataUrl` must additionally return the `version` value for each applicable ENS domain (or node) whose records are hosted on IPNS. This `version` value is incremented and then used by the gateway to publish new IPNS updates.

## Backwards Compatibility
`TBA`

## Security Considerations
`TBA`

## Copyright
Copyright and related rights waived via [CC0](../LICENSE.md).