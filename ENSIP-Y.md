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

#### Interpreting Metadata
The methods described in this section have been designed with autonomy, privacy, UI/UX and accessibility for end ENS users in mind. The plethora of off-chain storages have their own diverse ecosystems such that it in not uncommon for each storage to have its own set of UI/UX requirements, such as wallets, signer extensions etc. If users of ENS were to utilise such storage providers, they will inevitably be subjected to additional wallet extensions in their browsers. This is not ideal and the methods in this section have been crafted such that users do not need to install any additional UI/UX components or extensions other than their favourite ethereum wallet.

#### Key Generation
This draft proposes that both the `dataSigner` and `ipnsSigner` keypairs be generated **deterministically** from ethereum wallet signatures; see figure below.

![](https://raw.githubusercontent.com/namesys-eth/namesys-ccip-write/main/images/keygen.png)

This process involving deterministic key generation is implemented concisely in a single unified `keygen()` function in `namesys.js` library.

```ts
import { hkdf } from "@noble/hashes/hkdf";
import { sha256 } from "@noble/hashes/sha256";
import * as secp256k1 from "@noble/secp256k1";
import * as ed25519 from "@noble/ed25519";

/**
 * @param  username key identifier
 * @param    caip10 CAIP identifier for the blockchain account
 * @param signature Deterministic signature from X-wallet provider
 * @param  password Optional password
 * @returns Deterministic private/public keypairs as hex strings
 * Hex-encoded
 * [  ed25519.priv,   ed25519.pub],
 * [secp256k1.priv, secp256k1.pub]
 */
export async function keygen(
  username: string,
  caip10: string,
  signature: string,
  password: string | undefined
): Promise<[[string, string], [string, string]]> {
  // Signature must be at least of length 64
  if (signature.length < 64)
    throw new Error("SIGNATURE TOO SHORT; LENGTH SHOULD BE 65 BYTES");

  // Calulcate input key by hashing signature bytes using sha256 algorithm
  let inputKey = sha256(
    secp256k1.utils.hexToBytes(
      signature.toLowerCase().startsWith("0x") ? signature.slice(2) : signature
    )
  );

  // Calculate info from CAIP-10 identifier and username
  let info = `${caip10}:${username}`;

  // Calculate salt for keygen by hashing concatenated info, password and hex-encoded signature using sha256 algorithm
  let salt = sha256(
    `${info}:${password ? password : ""}:${signature.slice(-64)}`
  );

  // Calculate hash key output by feeding input key, salt and info to the HMAC-based key derivation function
  let hashKey = hkdf(sha256, inputKey, salt, info, 42);

  // Convert hash key to a private scalar for ed25519 elliptic curve
  let ed25519priv = ed25519.utils
    .hashToPrivateScalar(hashKey)
    .toString(16)
    .padStart(64, "0"); // ed25519 Private Key

  // Get public key by evaluating private scalar over ed25519 elliptic curve
  let ed25519pub = secp256k1.utils.bytesToHex(
    await ed25519.getPublicKey(ed25519priv)
  ); // ed25519 Public Key

  // Convert hash key to a private key for secp256k1 elliptic curve
  let secp256k1priv = secp256k1.utils.bytesToHex(
    secp256k1.utils.hashToPrivateKey(hashKey)
  ); // secp256k1 Private Key

  // Get public key by evaluating private key over secp256k1 elliptic curve
  let secp256k1pub = secp256k1.utils.bytesToHex(
    secp256k1.getPublicKey(secp256k1priv)
  ); // secp256k1 Public Key

  // Return both ed25519 and secp256k1 key types for IPNS and ethereum signers respectively
  return [
    // Hex-encoded [[ed25519.priv, ed25519.pub], [secp256k1.priv, secp256k1.pub]]
    [ed25519priv, ed25519pub],
    [secp256k1priv, secp256k1pub],
  ];
}
```

This `keygen()` function requires four variables: `caip10`, `username`, `password` and `signature`. Only `password` needs to be prompted from the user by the client; this field allows users to switch their IPNS namespace in the future. Both `caip10` and `username` are auto-derived from the connected wallet and/or the ENS domain undergoing records update. In the next section, derivation of the remaining `signature` parameter is described in detail.

```js
// CAIP-10 identifier
const caip10 = `eip155:${chainId}:${walletAddress}`;
```
```js
// Username is dependent on the storage type which can be 'WalletType' or
// 'DomainType'. See definitions at the end of this section.
let username;
if (storage === 'WalletType') username = `eth:${walletAddress}`;
if (storage === 'DomainType') username = ens;
```
```js
// IPNS secret key identifier; clients must prompt the user for this
const password = 'key1';
```

#### Deterministic Signatures
Deterministic signatures form the backbone of secure, keyless, autonomous and smooth UI when off-chain storages are in the mix. In the simplest implementation, at least two separate signatures need to be prompted from the users by the clients: `SIG_IPNS` & `SIG_SIGNER`.

##### a. `SIG_IPNS` for IPNS Keygen
`SIG_IPNS` is the deterministic ethereum signature responsible for IPNS key generation and for interpreting `ipnsSigner` metadata. Message payload for `SIG_IPNS` must be formatted as:

```text
Requesting Signature To Generate IPNS Key\n\nOrigin: ${username}\nKey Type: ${keyType}\nExtradata: ${extradata}\nSigned By: ${caip10}
```

##### b. `SIG_SIGNER` for Signer Keygen
`SIG_SIGNER` is the deterministic ethereum signature responsible for universal signer key generation. In order to enable batch records, a universal signer must be derived from the owner or manager keys of an ENS name. This signer is tasked with interpreting `dataSigner` metadata. Message payload for `SIG_SIGNER` must be formatted as:

```text
Requesting Signature To Generate ENS Records Signer\n\nOrigin: ${username}\nKey Type: ${keyType}\nExtradata: ${extradata}\nSigned By: ${caip10}
```

In both `SIG_IPNS` and `SIG_SIGNER` signature payloads, the `extradata` is calculated as

```solidity
// Calculating extradata in keygen signatures
bytes32 extradata = keccak256(
    abi.encodePacked(
        keccak256(
        abi.encodePacked(password)
        walletAddress
        )
    )
);
```

and `keyType` is currently `ed25519` for `SIG_IPNS` as required by IPNS and `secp256k1` for `SIG_SIGNER` since it is an ethereum signer. In the future, IPFS network plans to phase in `secp256k1` key types at which point `ed25519` key derivation won't be necessary.

With these deterministic formats for signature message payloads, the client must prompt the user for two `eth_sign` signatures. Once the user signs the messages, the `keygen()` function can derive the IPNS keypair and the signer keypair. The clients must additionally derive the IPNS CID and ethereum address corresponding to the IPNS and signer public keys using `processPubkeys()` function available in the `namesys.js` library. The metadata interpretation concludes with the client ensuring that 

- the derived IPNS CID must match the `ipnsSigner` metadata, and
- the derived signer's address must match the `dataSigner` metadata.

If these conditions are not met, clients must throw an error and inform the user of failure in interpretation of the metadata. If these conditions are met, then the client has the correct private keys to update a user's IPNS record as well as sign a user's ENS records for later verification by CCIP-Read. Since the derived signer can sign multiple records in the background without prompting the user, it is possible to update multiple records simultaneously with this method.

#### Storage Types
Storage types refer to two types of IPNS namespaces that can host a user's ENS records. In the first case of `DomainType`, each ENS domain has a unique IPNS container whose CID is stored in `ipnsSigner` metadata. In the second case of `WalletType`, a user can store the records for **all** the ENS domains owned or managed by a given wallet. Naturally, the second method is highly cost effective although it compromises on security to some extent; this is due to a single IPNS signer manifesting as a single point of compromise for records of all ENS domains in a wallet. This feature is achieved by choosing an appropriate `username` in the signature message payload of `SIG_IPNS` depending on the desired storage type. Similar cost-effectiveness can be achieved for the `dataSigner` metadata as well by choosing `WalletType` over `DomainType` when deriving `SIG_SIGNER`.

### Revert `StorageHandledByDatabase()`
The case of `StorageHandledByDatabase()` handler is a subset of the decentralised storage handler, in the sense that the clients should simply skip interpreting IPNS related metadata. This avoids having to derive `SIG_IPNS` and there is no concept of storage types for off-chain database handlers. Other than that, the entire process is the same as `StorageHandledByIPNS()`.
 
### Off-Chain Signers
It is possible to further save on gas costs by **not** storing the `dataSigner` metadata on chain. Services or users can instead post an approval for the `dataSigner` signed by the owner or manager of a domain along with the off-chain records. CCIP-Read can then verify this approval during resolution time and no on-chain `dataSigner` needs to be saved. This additional saving comes at the cost of one additional approval signature `SIG_APPROVAL` that the clients must prompt from the user. This signature must have the following message payload format:

```text
Requesting Signature To Approve ENS Records Signer\n\nOrigin: ${ens}\nApproved Signer: ${dataSigner}\nApproved By: ${caip10}
```

### Record Signatures: `SIG_RECORD`
Signature(s) `SIG_RECORD` accompanying the off-chain records must implement the following format in their message payloads:  

```text
Requesting Signature To Update ENS Record\n\nOrigin: ${ens}\nRecord Type: ${recordType}\nExtradata: ${extradata}\nSigned By: ${caip10}
```

where `extradata` must be calculated as follows,

```solidity
// Extradata in record signatures
bytes memory recordBytes = abi.encodePacked([dataType, recordValue])
bytes32 extradata = utils.bytesToHexString(
    abi.encodePacked(
        keccak256(
            recordBytes
        )
    )
);
```

where, 

- the `recordType` parameters are defined in ENSIP-6 and ENSIP-9, e.g. `text/avatar`, `address/60` etc, depending on each record, and 
- the `dataType` is simply the solidity data type of the record value.

### CCIP-Read Compatible Payload
The final `data:` payload in the off-chain record file could then follow this format,

```js
let encodedRecord = ethers.utils.defaultAbiCoder.encode([dataType], [recordValue]);
let encodeWithSelector = interface.encodeFunctionData("signedRecord", [
    dataSigner, // type 'address'
    SIG_RECORD, // type 'bytes'
    SIG_APPROVAL, // type 'bytes'
    encodedRecord // dynamic type
]);
```

which the CCIP-Read-enabled resolvers should first correctly decode, and then verify signer approval and record signatures, before resolving the record value.

## Backwards Compatibility
`TBA`

## Security Considerations
`TBA`

## Copyright
Copyright and related rights waived via [CC0](../LICENSE.md).