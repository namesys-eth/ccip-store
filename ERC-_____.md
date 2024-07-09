---
ERC: "5559"
TITLE: "The Off-Chain Data Write Protocol"
DESCRIPTION: Cross-chain write deferral protocol incorporating secure write deferrals to generic L1s, Ethereum L2s, centralised databases and decentralised & mutable storages
AUTHOR: (@sshmatrix), (@0xc0de4c0ffee), (@arachnid)
DISCUSSIONS-TO: ◥
STATUS: Draft
TYPE: Standards Track
CATEGORY: ERC
CREATED: ◥
REQUIRES: ◥ 
---

## Abstract
The following proposal is a superseding version of EIP-5559: Off-Chain Write Deferral Protocol, targeting wider set of storage types and introducing security measures to consider for secure off-chain write deferrals and subsequent retrievals. EIP-5559 in its present form is limited to deferring write operations to L2 EVM chains and centralised databases. Methods in this updated version enable secure write deferrals to generic L1s, Ethereum L2s, centralised databases and decentralised storages - mutable or immutable - such as IPFS, Arweave, Swarm etc. This draft alongside EIP-3668 is a significant step toward a complete and secure infrastructure for off-chain data retrieval and write deferral.

## Motivation
EIP-3668, or 'CCIP-Read' in short, has been key to retrieving off-chain data for a variety of contracts on Ethereum blockchain, ranging from price feeds for DeFi contracts, to more recently records for ENS users. The latter case is more interesting since it dedicatedly uses off-chain storage to bypass the usually high gas fees associated with on-chain storage; this aspect has a plethora of use cases well beyond ENS records and a potential for significant impact on universal affordability and accessibility of Ethereum. 

Off-chain data retrieval through EIP-3668 is a relatively simpler task since it assumes that all relevant data originating from off-chain storages is translated by CCIP-Read-compliant HTTP gateways; this includes L2 chains, centralised databases or decentralised storages. On the flip side however, so far each service leveraging CCIP-Read must handle two main tasks externally:

- writing this data securely to these storage types on their own, and

- incorporating reasonable security measures in their CCIP-Read compatible contracts for verifying this data before performing on-chain read or write operations.

Writing to a variety of centralised and decentralised storages is a broader objective compared to CCIP-Read largely due to two reasons:

1. Each storage provider typically has its own architecture that the write operation must comply with, e.g. they may require additional credentials and configurations to able to write data to them, and

2. Each storage must incorporate some form of security measures during write operations so that off-chain data's integrity can be verified by CCIP-Read contracts during data retrieval stage.

EIP-5559 was the first step toward such a tolerant 'CCIP-Write' protocol which outlined how write deferrals could be made to L2 and centralised databases. The cases of L2 and database are similar; deferral to an L2 involves routing the `eth_call` to L2, while deferral to a database can be made by extracting `eth_sign` from `eth_call` and posting the resulting signature along with the data for later verification. In both cases, no pre-flight information needs to be processed by the client and arguments of `eth_call` and `eth_sign` as specified in the current EIP-5559 are sufficient. This proposal supersedes the previous EIP-5559 by re-introducing secure write deferrals to generic L1s, EVM L2s, databases and decentralised storages, especially those which - beyond the arguments of `eth_call` and `eth_sign` - require additional pre-flight metadata from clients to successfully host users' data on their favourite storage. This document also enables more complex and generic use-cases of storages such as those which do not store the signers' addressess on chain as presumed in the current EIP-5559.

### Curious Case of Decentralised Storages
Decentralised storages powered by cryptographic protocols are unique in their diversity of architectures compared to centralised databases or L2 chains, both of which have canonical architectures in place. For instance, write calls to L2 chains can be generalised through the use of `chainId` for any given `callData`; write deferral in this case is as simple as routing the `eth_call` to another contract on an L2 chain. There is no need to incorporate any additional security requirement(s) since the L2 chain ensures data integrity locally, while the global integrity can be proven by employing a state verifier scheme (e.g. EVM-Gateway) during CCIP-Read calls. Same argument applies to generic L1 blockchains as well. Centralised databases have a very similar architecture where instead of invoking `eth_call`, the result of `eth_sign` needs to be posted on the database along with the `callData` for integrity verification by CCIP-Read.

Decentralised storages on the other hand, do not typically have EVM- or database-like environments and may have their own unique content addressing requirements. For example, IPFS, Arweave, Swarm etc all have unique content identification schemes as well as their own specific fine-tunings and/or choices of cryptographic primitives, besides supporting their own cryptographically secured namespaces. This significant and diverse deviation from EVM-like architecture results in an equally diverse set of requirements during both the write deferral operation as well as the subsequent state verifying stage.

For example, consider a scenario where the choice of storage is IPNS or ArNS. In precise terms, IPNS storage refers to immutable IPFS content wrapped in mutable IPNS namespace, which eventually serves as the reference coordinate for off-chain data. The case of ArNS is similar; ArNS is immutable Arweave content wrapped in mutable ArNS namespace. To write to IPNS or ArNS storage, the client requires more information than only the gateway URL responsible for write operations and arguments of `eth_sign`. More precisely, the client must at least prompt the user for their IPNS or ArNS signature which is necessary for updating the namespaced storage. The client may also need additional information from the user such as specific arguments required by IPNS or ArNS signature. One such example is the requirement of encoded `version` of IPNS update which goes into the construction of IPNS record payload. These additional user-centric requirements are not accommodated by EIP-5559 in its present form, and the resolution of these issues is detailed in the following attempt towards a suitable CCIP-Write specification.

## Specification
### Overview
The following specification revolves around the structure and description of an arbitrary off-chain storage handler tasked with the responsibility of writing to an arbitrary storage. First introduced in EIP-5559, the protocol outlined herein re-defines the construction of the `StorageHandledBy__()` revert to accept generic L1 blockchains, EVM L2s, databases and decentralised & namespaced storages. In particular, this draft proposes that `StorageHandledByL2()` and `StorageHandledByOffChainDatabase()` introduced in EIP-5559 be replaced with re-defined `StorageHandledByL2()` and `StorageHandledByDatabase()` respectively, and new `StorageHandledBy__()` reverts be allowed through new EIPs that sufficiently detail their interfaces and designs. Some foreseen examples of new storage handlers include `StorageHandledBySolana()` for Solana, `StorageHandledByFilecoin()` for Filecoin, `StorageHandledByIPFS()` for IPFS, `StorageHandledByIPNS()` for IPNS, `StorageHandledByArweave()` for Arweave, `StorageHandledByArNS()` for ArNS, `StorageHandledBySwarm()` for Swarm etc.

![](https://raw.githubusercontent.com/namesys-eth/namesys-ccip-write/main/images/schematic.png)

Similar to EIP-5559, a CCIP-Write deferral call to an arbitrary function `setValue(bytes32 key, bytes32 value)` can be described in pseudo-code as follows:

```solidity
// Define revert event
error StorageHandledBy__(address sender, bytes callData, bytes metadata);

// Generic function in a contract
function setValue(
    bytes32 key,
    bytes32 value
) external {
    // Get metadata from on-chain sources
    bytes metadata = getMetadata(key);  
    // Defer write call to off-chain handler
    revert StorageHandledBy__(
        msg.sender, 
        abi.encode(key, value), 
        metadata
    );
};
```

where the following structure for `StorageHandledBy__()` has been followed:

```solidity
// Details of revert event
error StorageHandledBy__(
    address msg.sender, // Sender of call
    bytes callData, // Payload to store
    bytes metadata // Metadata required by off-chain clients
);
```

#### Metadata
The arbitrary `metadata` field captures all the relevant information that the client may require to update a user's data on their favourite storage. For instance, `metadata` must contain a pointer to a user's data on their desired storage. In the case of `StorageHandledByL2()` for example, `metadata` must contain a chain identifier such as `chainId` and additionally the contract address. In case of `StorageHandledByDatabase()`, `metadata` must contain the custom gateway URL serving a user's data. In case of `StorageHandledByIPNS()`, `metadata` may contain the public key of a user's IPNS container; the case of ArNS is similar. In addition, `metadata` may further contain security-driven information such as a delegated signer's address who is tasked with signing the off-chain data; such signers and their approvals must also be contained for verification tasks to be performed by the client. It is left up to each storage handler `StorageHandledBy__()` to precisely define the structure of `metadata` in their documentation for the clients to refer to. This proposal introduces the structure of `metadata` for four storage handlers: Solana L1, EVM L2s, databases and IPNS as follows.

### Solana Handler: `StorageHandledBySolana()`
A Solana storage handler simply requires the hex-encoded `programId` and the manager `account` on the Solana blockchain; `programId` is equivalent to a contract address on Solana. Since Solana natively uses `base58` encoding in its virtual machine setup, `programId` values must be hex-encoded according to EIP-2308 for storage on Ethereum. These hex-encoded values in the `metadata` must eventually be decoded back to `base58` for usage on Solana. 

```solidity
// Revert handling Solana storage handler
error StorageHandledBySolana(address sender, bytes callData, bytes metadata);

(
    bytes32 programId, // Program (= contract) address on Solana; hex-encoded
    bytes32 account // Manager account on Solana; hex-encoded
) = getMetadata(node); // Arbitrary code
// programId = 0x37868885bbaf236c5d2e7a38952f709e796a1c99d6c9d142a1a41755d7660de3
// account = 0xe853e0dcc1e57656bd760325679ea960d958a0a704274a5a12330208ba0f428f
bytes metadata = abi.encode(programId, account);
bytes callData = abi.encode(node, key, value);
address sender = msg.sender;
```

Clients implementing the Solana handler must call the Solana `programId` using a Solana wallet that is connected to `account` as follows. 

```js
/* Pseudo-code to write to Solana program (= contract) */
// Instantiate program interface on Solana
const program = new program(programId, rpcProvider);
// Connect to Solana wallet
const wallet = useWallet();
// Decode off-chain data from encoded calldata in revert
let [node, key, value] = abi.decode(callData);
// Call the Solana program using connected wallet with off-chain data
// [!] Only approved manager in the Solana program should call
if (wallet.publicKey === account === program.isManagerFor(account, msg.sender)) {
    await program(wallet).setValue(node, key, value);
}
```

In the above example, `programId`, `account` and `msg.sender` must be decoded to `base58` from `hex`. Solana handler requires a one-time transaction on Solana during initial setup for each user to set the local manager. This call in form of pseudo-code is simply 

```js 
await program(wallet).setManagerFor(account, msg.sender)
```

### L2 Handler: `StorageHandledByL2()`
A minimal L2 handler only requires the list of `chainId` values and the corresponding `contract` addresses, although additional measures must be taken to ensure that the integrity of calldata is maintained by the gateway before the call is routed to L2. Therefore, this proposal formalises that

1. The storage handler must contain a `proof` of the original calldata. This `proof` is simply

    ```solidity
    // Proof is hash of concatenated calldata
    bytes32 proof = keccak256(
        abi.encodePacked(
            bytes32 key,
            bytes32 value
        )
    )
    ```

    for any generic function `function setValue(bytes32 key, bytes32 value) external` in the L1 contract.

2. The L2 contract must implement `function setValueWithProof(bytes32 key, bytes32 value, bytes32 proof) external`

3. The deferral in this case must prompt the client to build the transaction with original calldata and the proof, and submit it to the L2 by calling the `setValueWithProof()` function.

4. The `setValueWithProof()` function must internally calculate the proof, match it with the proof provided by the gateway and revert if the proofs do not match.
 
One example construction of an L2 handler in an L1 contract is given below.

#### L1
```solidity
// Define revert event
error StorageHandledByL2(
    address sender, 
    address contractL2, 
    uint256 chainId, 
    bytes32 proof
);

// Generic function in a contract
function setValue(
    bytes32 node,
    bytes32 key,
    bytes32 value
) external {
    // Get metadata from on-chain sources
    (
        address contractL2, // Contract address on L2
        uint256 chainId // L2 ChainID
    ) = getMetadata(node); // Arbitrary code
    // contract = 0x32f94e75cde5fa48b6469323742e6004d701409b
    // chainId = 21
    address sender = msg.sender;
    bytes32 proof = keccak256(
        abi.encodePacked(
            bytes32 node,
            bytes32 key,
            bytes32 value
        )
    )
    // Defer write call to off-chain handler
    revert StorageHandledByL2(
        msg.sender, 
        contractL2,
        chainId,
        proof
    );
};
```

#### L2
```solidity
// Function in L2 contract
function setValueWithProof(
    bytes32 node,
    bytes32 key,
    bytes32 value,
    bytes32 proof
) external {
    // Calculate proof
    bytes32 _proof = keccak256(
        abi.encodePacked(
            bytes32 node,
            bytes32 key,
            bytes32 value
        )
    )
    // Verify intergity of calldata
    require(_proof === proof, "BAD_PROOF");
    ... // Rest of the code
}
```

### Database Handler: `StorageHandledByDatabase()`
A minimal database handler is similar to an L2 in the sense that:

  a) similar to `chainId`, it requires the `gatewayUrl` that is tasked with handling off-chain write operations, and

  b) similar to `eth_call`, it must require `eth_sign` output to secure the data, and the client must prompt the users for these signatures.

In this case, the `metadata` must contain the bespoke `gatewayUrl` and may additionally contain the addresses of `dataSigner` of `eth_sign`. If a `dataSigner` is included in the metadata, then the client must make sure that the signature forwarded to the gateway is signed by that `dataSigner`. It is possible for the `dataSigner` to exist off-chain instead and not be returned in the metadata; for this scenario, refer to additional details in section 'Off-Chain Signers'. One example construction of a database handler's `metadata` is given below.

```solidity
error StorageHandledByDatabase(address sender, bytes callData, bytes metadata);

(
    string gatewayUrl, // Gateway URL
    address dataSigner // Ethereum signer's address; must be address(0) for off-chain signer
) = getMetadata(node);
// gatewayUrl = "https://api.namesys.xyz"
// dataSigner = 0xc0ffee254729296a45a3885639AC7E10F9d54979
bytes metadata = abi.encode(gatewayUrl, dataSigner);
bytes callData = abi.encode(node, key, value);
address sender = msg.sender;
```

In the above example, the client must first verify that the `eth_sign` is signed by a matching `dataSigner`, then prompt the user for a signature and finally pass the resulting signature to the `gatewayUrl` along with the off-chain data. The message payload for this signature must be formatted according to the directions in the 'Data Signatures' section further down this document. The off-chain data and the signatures must be encoded according to the directions in the 'CCIP-Read Compatiable Payload' section. Further directions for precise handling of the message payloads and metadata for databases are provided in 'Interpreting Metadata' section.

### Decentralised Storage Handler: `StorageHandledByIPNS()`
Decentralised storages are the extremest in the sense that they come both in immutable and mutable form; the immutable forms locate the data through immutable content identifiers (CIDs) while mutable forms utilise some sort of namespace which can statically reference any dynamic content. Examples of the former include raw content hosted on IPFS and Arweave while the latter forms use IPNS and ArNS namespaces respectively to reference the raw and dynamic content. 

The case of immutable forms is similar to a database although these forms are not as useful in practise so far. This is due to the difficulty associated with posting the unique CID on chain each time a storage update is made. One way to bypass this difficulty is by storing the CID cheaply in an L2 contract; this method requires the client to update the data on both the decentralised storage as well as the L2 contract through two chained deferrals. CCIP-Read in this case is also expected to read from two storages to be able to fully handle a read call. Contrary to this tedious flow, namespaces can instead be used to statically fetch immutable CIDs. For example, instead of a direct reference to immutable CIDs, IPNS and ArNS public keys can instead be used to refer to IPFS and Arweave content respectively; this method doesn't require dual deferrals by CCIP-Write (or CCIP-Read), and the IPNS or Arweave public key needs to be stored on chain only once. However, accessing the IPNS and ArNS content now requires that the client must prompt the user for additional information, e.g. IPNS and ArNS signatures in order to update the data.

Decentralised storage handlers' `metadata` structure is therefore expected to contain additional context which the clients must interpret and evaluate before calling the gateway with the results. This feature is not supported by EIP-5559 and services using EIP-5559 are thus incapable of storing data on decentralised namespaced & mutable storages. One example construction of a decentralised storage handler's `metadata` for IPNS is given below.

```solidity
error StorageHandledByIPNS(address sender, bytes callData, bytes metadata);

(
    string gatewayUrl, // Gateway URL for POST-ing
    address dataSigner, // Ethereum signer's address; must be address(0) for off-chain signer
    bytes ipnsSigner // Context for namespace (IPNS signer's hex-encoded CID)
) = getMetadata(node);
// gatewayUrl = "https://ipns.namesys.xyz"
// dataSigner = 0xc0ffee254729296a45a3885639AC7E10F9d54979
// ipnsSigner = 0xe50101720024080112203fd7e338b2de90159832ffcc434927da8bbfc3a000fa58ea0548aa8e08f7e10a
bytes metadata = abi.encode(gatewayUrl, dataSigner, ipnsSigner);
bytes callData = abi.encode(node, key, value);
address sender = msg.sender;
```

In the example above, a client must evaluate the metadata according to the following outline. The client must request the user for an IPNS signature verifiable against the IPNS CID returned in the `ipnsSigner` metadata. If verified, the client will then additionally require the historical context encoding the previous IPNS record's `version` data (e.g. sequence number, validity etc) to make the IPNS update. There are several ways of providing this `version` data to the clients, e.g. a dedicated API, IPFS Pub/Sub, L2 indexer etc. It is therefore left up to individual implementations and/or protocols to choose their own desired method for `version` indexing, e.g. ENSIP-16/-19 for ENS. With `version` data in hand, the client can move on to signing the off-chain data using the `dataSigner` key. Once signed, the client must encode the off-chain data, the data signature and the approval signature in a CCIP-Read-compatible payload. Lastly, the client must: 

- calculate the IPFS hash corresponding to the new off-chain data payload,
- increment the IPNS record by encoding the `version` with new IPFS hash, and
- broadcast the IPNS update by signing and publishing the incremented `version` to the `gatewayurl`. 

The strictly typed formatting for this IPNS signature payload is internally handled by the IPNS protocol and IPNS service providers via standard libraries. 

### Interpreting Metadata
The following section describes the precise interpretation of the metadata common to both IPNS and database storage handlers. The methods described in this section have been designed with autonomy, privacy, UI/UX and accessibility for ethereum users in mind. The plethora of off-chain storages have their own diverse ecosystems such that it in not uncommon for each storage to have its own set of UI/UX requirements, such as wallets, signer extensions etc. If ethereum users were to utilise such storage providers, they will inevitably be subjected to additional wallet extensions in their browsers. This is not ideal and the methods in this section have been crafted such that users do not need to install any additional UI/UX components or extensions other than their favourite ethereum wallet. 

`StorageHandledByIPNS()` is more complex in construction than `StorageHandledByDatabase()` which is a reduced version of the former. For this reason, we still start by describing how clients must implement `StorageHandledByIPNS()` first. Later on, we will reduce the requirements to the simpler case of `StorageHandledByDatabase()`.

#### Key Generation
This draft proposes that both the `dataSigner` and `ipnsSigner` keypairs be generated deterministically from ethereum wallet signatures; see figure below.

![](https://raw.githubusercontent.com/namesys-eth/namesys-ccip-write/main/images/keygen.png)

This process involving deterministic key generation can be implemented concisely in a single unified `keygen()` function as follows.

```js
/* Pseudo-code for key generation */
function keygen(
  username, // Key identifier
  caip10, // CAIP identifier for the blockchain account
  signature, // Deterministic signature from wallet
  password // Optional password
) {
  // Calculate input key by hashing signature bytes using SHA256 algorithm
  let inputKey = sha256(signature);
  // Calculate info from CAIP-10 identifier and username
  let info = `${caip10}:${username}`;
  // Calculate salt for keygen by hashing concatenated info, hashed password and hex-encoded signature using SHA256 algorithm
  let salt = sha256(`${info}:${sha256(password || "")}:${signature}`);
  // Calculate hash key output by feeding input key, salt & info to the HMAC-based key derivation function (HKDF) with dLen = 42
  let hashKey = hkdf(sha256, inputKey, salt, info, 42);
  // Calculate and return both ed25519 and secp256k1 keypairs
  return [
    ed25519(hashKey), // Calculate ed25519 keypair from hash key
    secp256k1(hashKey) // Calculate secp256k1 keypair from hash key
  ]
}
```

This `keygen()` function requires four variables: `caip10`, `username`, `password` and `signature`. Their descriptions are given below.

##### 1. `caip10`
CAIP-10 identifier `caip10` is auto-derived from the connected wallet's checksummed address `wallet` and `chainId`.

```js
/* CAIP-10 identifier */
const caip10 = `eip155:${chainId}:${wallet}`
```

##### 2. `username`
`username` may be prompted from the user by the client or determined by the protocol. This public field allows users to switch their protocol-specific IPNS namespace in the future. For instance, protocols may set `username` deterministically as equal to `caip10` or some protocol-specific function of `node`; see example below.

```js
/* Username is dependent on the storage type which can be 'walletType' or 'nodeType'. See definitions at the end of this section */
// Example: node = namehash(normalise(ens)) for ENS, aka preimage(node) = ens
let username;
if (storage === 'walletType') username = caip10;
if (storage === 'nodeType') username = preimage(node);
```

##### 3. `password`
`password` is an optional private field and it must be prompted from the user by the client; this field allows users to secure their IPNS namespace for a given `username`.
```js
/* IPNS secret key identifier */ 
// Clients must prompt the user for this
const password = 'key1'
```

##### 4. `signature`
Deterministic signature forms the backbone of secure, keyless, autonomous and smooth UI when off-chain storages are in the mix. In the simplest implementation, one such signature must be prompted from the users by the clients. `sigKeygen` is the deterministic ethereum signature responsible for 

- the IPNS key generation and for interpreting `ipnsSigner` metadata, and
- the delegated signer key generation and for interpreting `dataSigner` metadata. 

In order to enable batch data writing for multiple nodes, a delegated signer must be derived from the owner or manager keys of a node. Message payload for `sigKeygen` must be formatted as:

```text
Requesting Signature To Generate Keypair(s)\n\nOrigin: ${username}\nProtocol: ${protocol}\nExtradata: ${extradata}\nSigned By: ${caip10}
```

where the `extradata` is calculated as follows,

```solidity
// Calculating extradata in keygen signatures
bytes32 extradata = keccak256(
    abi.encodePacked(
        pbkdf2(
            password, 
            salt, 
            iterations
        ), // Stretch password with PBKDF2
        wallet
    )
)
```

where `PBKDF2` - with `keccak256(abi.encodePacked(username))` as salt and last 5 hex-nibbles converted to `uint` as the iteration count - is used for brute-force vulnerability protection.

```js
/* Definitions of salt and iterations in PBKDF2 */
let salt = keccak256(abi.encodePacked(username));
let iterations = uint(salt.slice(-5)); // max(iterations) = uint(0xFFFFF) = 1048757
```

The remaining `protocol` field is a protocol-specific identifier limiting the scope to a specific protocol. This identifier cannot be global and must be uniquely defined by each implementation or protocol. With this deterministic format for signature message payload, the client must prompt the user for `eth_sign` signature. Once the user signs the messages, the `keygen()` function can derive the IPNS keypair and the signer keypair. The clients must additionally derive the IPNS CID and ethereum address corresponding to the IPNS and signer public keys. The metadata interpretation concludes with the client ensuring that 

- the derived IPNS CID must match the `ipnsSigner` metadata, and
- the derived signer's address must match the `dataSigner` metadata.

If these conditions are not met, clients must throw an error and inform the user of failure in interpretation of the metadata. If these conditions are met, then the client has the correct private keys to update a user's IPNS record as well as sign a user's data for later verification by CCIP-Read. Since the derived signer can sign multiple instances of off-chain data in the background without prompting the user, it is possible to update data for multiple nodes simultaneously with this method.

#### Storage Types
Storage types refer to two types of IPNS namespaces that can host a user's data. In the first case of `nodeType`, each `node` has a unique IPNS container whose CID is stored in `ipnsSigner` metadata. In the second case of `walletType`, a user can store the data for all nodes owned or managed by a given wallet. Naturally, the second method is highly cost effective although it compromises on security to some extent; this is due to a single IPNS signer manifesting as a single point of compromise for all off-chain data for a wallet. This feature is achieved by choosing an appropriate `username` in the signature message payload of `sigKeygen` depending on the desired storage type.

During the initialisation step when the user sets on-chain `ipnsSigner` for the first time, the clients must prompt the user for their choice of storage type. Depending on the user's choice, IPNS CID can be posted on chain with an appropriate index.

```js
/* Setting IPNS signer on-chain during initialisation setup */
// IPNS signer derived from keygen() in CIDv1 format
let cid = 'bafyreibcli3vlmr4et6oekv3xdjx2sm6k4tioynbavmwgrsevklujpzywu';
// IPNS signer is function of node for 'nodeType' storage; remove constant 'e5010172002408011220' prefix from hex-encoded payload to save gas
if (storage === 'nodeType') setIpnsSigner(node, cid.encode('hex').replace('e5010172002408011220', ''));
// IPNS signer is function of wallet for 'walletType' storage; remove constant 'e5010172002408011220' prefix from hex-encoded payload to save gas
if (storage === 'walletType') setIpnsSigner(bytes32(uint256(uint160(wallet))), cid.encode('hex').replace('e5010172002408011220', ''));
```

CCIP-Write-enabled contracts should implement an appropriate internal mechanism for fetching IPNS signer as a function of `node` or `wallet`. This mechanism must follow the previously mentioned fallback strategy: the contract must first check if `nodeType` storage exists for a given `node`, and if no `ipnsSigner` exists for a `node`, then the contract should check for fallback `walletType` storage for a `wallet` and return the result in the revert.

### Revert `StorageHandledByDatabase()`
The case of `StorageHandledByDatabase()` handler is a subset of the decentralised storage handler, in the sense that the clients should simply skip interpreting IPNS related metadata. There is additionally no concept of storage types for off-chain database handlers. Other than that, the entire process is the same as `StorageHandledByIPNS()`.
 
### Off-Chain Signers
It is possible to further save on gas costs by not storing the `dataSigner` metadata on chain. In detail, instead of storing the `dataSigner` on chain for verification, clients can provide the user with the option to,

- request an approval for an off-chain `dataSigner` signed by the owner or manager of a node, and
- post this approval and the off-chain `dataSigner` along with the off-chain data in encoded form.

CCIP-Read-enabled contracts can then verify during resolution time that the approval attached with the data comes from the node's manager or owner and that it approves the expected `dataSigner`. Using this mechanism of delegating signatures to an off-chain signer, no on-chain `dataSigner` needs to be posted. This additional saving comes at the cost of one additional approval signature `approval` that the clients must prompt from the user. This signature must have the following message payload format:

```text
Requesting Signature To Approve Data Signer\n\nOrigin: ${username}\nApproved Signer: ${dataSigner}\nApproved By: ${caip10}
```

where `dataSigner` must be checksummed.

### Data Signatures
Signature(s) `sigData` accompanying the off-chain data must implement the following format in their message payloads:  

```text
Requesting Signature To Update Off-Chain Data\n\nOrigin: ${username}\nData Type: ${dataType}\nData Value: ${dataValue}
```

where `dataType` parameters are protocol-specific; they are defined in ENSIP-5, ENSIP-7 and ENSIP-9 for ENS (formerly EIP-634, EIP-1577 and EIP-2308 respectively), e.g. `text/avatar`, `address/60` etc.

### CCIP-Read Compatible Payload
The final EIP-3668-compatible `data` payload in the off-chain data file must then follow this format,

```solidity
bytes encodedData = abi.encode(['bytes'], [dataValue])
bytes dataPayload = abi.encode(
    ['address', 'bytes32', 'bytes32', 'bytes'],
    [dataSigner, sigData, approval, encodedData]
)
```

which the CCIP-Read-enabled contracts must first correctly decode, and then verify signer approval and data signatures, before resolving the data value. The client must construct this `data` and pass it to the gateway in the `POST` request along with the raw values for indexing.

### `POST` & Protocol-specific Parameters
For any storage other than a blockchain with a wallet extension, the client must call the `gatewayUrl` via a `POST` request. The structure of the `POST` is protocol-specific and left up to individual protocols to handle internally. Besides the `POST` request, `username`, `protocol` and `dataType` are the other protocol-specific parameters that we have encountered in the text before. Note that we didn't yet define the paths for the off-chain data files either, i.e. where should the file containing off-chain `data` be stored and later referred to in CCIP-Read-compatible contracts? These `path` schemes are also native to each implementation and are therefore left up to each protocol to define along with the previously mentioned parameters. The combined total of five parameters should be defined by the protocols through a native improvement proposal. For example, `POST` format, `username`, `protocol`, `dataType`, and `path` for ENS are described in ENSIP-19. 

### New Revert Events
1. Each new storage handler must submit their `StorageHandledBy__()` identifier through an ERC track proposal referencing the current draft and EIP-5559.

2. Each `StorageHandledBy__()` provider must be supported with detailed documentation of its structure and the necessary `metadata` that its implementers must return.

3. Each `StorageHandledBy__()` proposal must define the precise formatting of any message payloads that require signatures and complete descriptions of custom cryptographic techniques implemented for additional security, accessibility or privacy.

## Implementation featuring ENS
ENS off-chain resolvers capable of reading from and writing to decentralised storages are perhaps the most complex use-case for CCIP-Read and CCIP-Write. One example of such a (minimal) resolver is given below along with the client-side code for handling the revert.

### Contract
```solidity
/* ENS resolver implementing StorageHandledByIPNS() */
interface iResolver {
    // Defined in EIP-5559
    error StorageHandledByIPNS(
        address sender,
        bytes callData,
        bytes metadata
    );
    // Defined in EIP-137
    function setAddr(bytes32 node, address addr) external;
}

// Defined in EIP-5559
string public gatewayUrl = "https://post.namesys.xyz"; // RESTful API endpoint
string public metadataUrl = "https://gql.namesys.xyz"; // GQL API endpoint

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
    // Get ethereum signer & IPNS CID stored on-chain with arbitrary logic/code
    // Both may be unique to each name, or each owner or manager address
    (address dataSigner, bytes ipnsSigner) = getMetadata(node); 
    // Construct metadata required by off-chain clients. Clients must refer to ENSIP-19 for directions to interpret this metadata
    bytes memory metadata = abi.encode(
        gatewayUrl, // Gateway URL tasked with writing to IPNS
        dataSigner, // Ethereum signer's address
        ipnsSigner, // IPNS signer's hex-encoded CID as context for namespace
        metadataUrl // GraphQL endpoint for encoded version (per ENSIP-16)
    )
    // Defer to IPNS storage
    revert StorageHandledByIPNS(
        msg.sender,
        abi.encode(node, addr),
        metadata
    );
}
```

### Client-side
```js
/* Client-side pseudo-code in ENS App */
// IPNS publishing provider
import IPNS from provider;
// Decode calldata from revert
const [node, addr] = abi.decode(callData);
// Decode metadata from revert
const [gatewayUrl, dataSigner, ipnsSigner, metadataUrl] = abi.decode(metadata);
// Fetch last IPNS version data from metadata API endpoint
let version = await fetch(metadataUrl, node);
// Deterministically generate IPNS and signer keypairs
let [ipnsKey, signerKey] = keygen(username, caip10, sigKeygen, password);
// Check if generated IPNS and signer public keys match the metadata
if (ipnsKey.pub === ipnsSigner && signerKey.pub === dataSigner) {
    // Sign the data with signer private key
    let signedData = await signData(node, addr, signerKey.priv);
    // Make IPFS content from signed data
    let ipfsCid = makeIpfs(signedData);
    // Create IPNS revision to publish from version data
    let revision = IPNS.v0(ipfsCid) || IPNS.increment(version, ipfsCid);
    // Publish revision to IPFS network
    await IPNS.publish(gatewayUrl, revision, signedData, ipnsKey.priv);
} else {
    // Tell user that derived keypairs did not match metadata
    throw Error('Bad Credentials');
}
```

## Backwards Compatibility
Methods in this document are not compatible with previous EIP-5559 specifications.

## Security Considerations
1. Since both the `ed25519` and `secp256k1` private keys for IPNS and delegated signer respectively are derived from the same `signature` and `hashKey`, leaking one key is equivalent to leaking the other. 

2. Clients must purge the derived IPNS and signer private keys from local storage immediately after signing the IPNS update and off-chain data respectively.

3. Signature message payload and the resulting deterministic signature `sigKeygen` must be treated as a secret by the clients and immediately purged from local storage after usage in the `keygen()` function.

4. Clients must immediately purge the `password` from local storage after usage in the `keygen()` function.

## Copyright
Copyright and related rights waived via [`CC0`](../LICENSE.md).