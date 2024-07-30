# Inbox Contract

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**
- [Inbox Contract](#inbox-contract)
  - [Motivation](#motivation)
  - [How It Works](#how-it-works)
  - [Supporting Dynamic Updates to Inbox Address](#supporting-dynamic-updates-to-inbox-address)
    - [SystemConfig](#systemconfig)
      - [setBatchInbox](#setbatchinbox)
      - [initialize](#initialize)
      - [UpdateType](#updatetype)
    - [How `op-node` knows the canonical batch inbox](#how-op-node-knows-the-canonical-batch-inbox)
    - [How `op-batcher` knows canonical batch inbox](#how-op-batcher-knows-canonical-batch-inbox)
  - [Upgrade](#upgrade)
  - [Reference Implementation](#reference-implementation)

## Motivation

The batch inbox is currently an Externally Owned Account (EOA), which has both advantages and disadvantages:

Advantages:
- Low submission gas cost due to the absence of onchain execution.
- Verification logic is moved offchain to the derivation part, protected by a fault dispute game with a correct absolute prestate.

Disadvantage:
- Onchain verification is not possible.


This specification aims to allow the batch inbox to be a contract, enabling customized batch submission conditions such as:
- Requiring the batch transaction to be signed by a quorum of sequencers in a decentralized sequencing network; or
- Mandating that the batch transaction call a BLOB storage contract (e.g., EthStorage) with a long-term storage fee, which is then distributed to data nodes that prove BLOB storage over time.


## How It Works

The integration process consists of three primary components:
1. Replacement of the [`BatchInboxAddress`](https://github.com/ethereum-optimism/optimism/blob/db107794c0b755bc38a8c62f11c49320c95c73db/op-chain-ops/genesis/config.go#L77) with an inbox contract: The existing `BatchInboxAddress`, which currently points to an Externally Owned Account (EOA), will be replaced by a smart contract. This new inbox contract will be responsible for verifying and enforcing batch submission conditions.
2. Modification of the `op-node` derivation process: The `op-node` will be updated to exclude failed batch transactions during the derivation process. This change ensures that only successfully executed batch transactions are processed and included in the derived state. 
3. Modification of the op-batcher submission process: The op-batcher will be updated to [call `recordFailedTx`](https://github.com/blockchaindevsh/optimism/blob/02e3b7248f1b590a2adf1f81488829760fa2ba03/op-batcher/batcher/driver.go#L537) for failed batch transactions. This modification ensures that the data contained in failed transactions will be resubmitted automatically.
   1. Most failures will be detected during the [`EstimateGas`](https://github.com/ethereum-optimism/optimism/blob/8f516faf42da416c02355f9981add3137a3db190/op-service/txmgr/txmgr.go#L266) call. However, under certain race conditions, failures may occur after the transaction has been included in a block.

These modifications aim to enhance the security and efficiency of the batch submission and processing pipeline, allowing for more flexible and customizable conditions while maintaining the integrity of the derived state.


## Supporting Dynamic Updates to Inbox Address

### SystemConfig

The `SystemConfig` is the source of truth for the address of inbox. It stores information about the inbox address and passes the information to L2 as well. 


#### setBatchInbox

A new function `setBatchInbox` is introduced to the `SystemConfig` contract, enabling dynamic updates to the inbox:

```solidity
/// @notice Updates the batch inbox address. Can only be called by the owner.
/// @param _batchInbox New batch inbox address.
function setBatchInbox(address _batchInbox) external onlyOwner {
    _setBatchInbox(_batchInbox);
}

/// @notice Updates the batch inbox address.
/// @param _batchInbox New batch inbox address.
function _setBatchInbox(address _batchInbox) internal {
    Storage.setAddress(BATCH_INBOX_SLOT, _batchInbox);

    bytes memory data = abi.encode(_batchInbox);
    emit ConfigUpdate(VERSION, UpdateType.BATCH_INBOX, data);
}

```


#### initialize

The `SystemConfig` now emits an event when the inbox is initialized, while retaining its existing support for inbox configuration during initialization.

```solidity
function initialize(
    address _owner,
    uint32 _basefeeScalar,
    uint32 _blobbasefeeScalar,
    bytes32 _batcherHash,
    uint64 _gasLimit,
    address _unsafeBlockSigner,
    ResourceMetering.ResourceConfig memory _config,
    address _batchInbox,
    SystemConfig.Addresses memory _addresses
)
    public
    initializer
{
    ...
    // Storage.setAddress(BATCH_INBOX_SLOT, _batchInbox);
    // initialize inbox by `_setBatchInbox` so that an event is emitted.
    _setBatchInbox(_batchInbox);
    ...
}
```

#### UpdateType

A new enum value `BATCH_INBOX` is added to the `UpdateType` enumeration.

```solidity
enum UpdateType {
    BATCHER,
    GAS_CONFIG,
    GAS_LIMIT,
    UNSAFE_BLOCK_SIGNER,
    BATCH_INBOX
}
```

### How `op-node` knows the canonical batch inbox

We define the canonical batch inbox at a specific L2 block(denoted as `bn`) as the batch inbox corresponding to the L1 block that serves as the origin of `bn`.

Under normal conditions, `op-node` knows the canonical batch inbox through the derivation pipeline:
1. The `L1Traversal` component first identifies the L1 `SystemConfig` changes while traversing the L1 block, via [`UpdateSystemConfigWithL1Receipts`](https://github.com/ethereum-optimism/optimism/blob/71928829ca7ece48152159daa1d231eac2df03b3/op-node/rollup/derive/l1_traversal.go#L78).
   1. The [`ProcessSystemConfigUpdateLogEvent`](https://github.com/ethereum-optimism/optimism/blob/71928829ca7ece48152159daa1d231eac2df03b3/op-node/rollup/derive/system_config.go#L59) function will be modified to parse the newly added inbox change.
2. The `L1Retrieval` component then fetches the canonical batch inbox from the `L1Traversal` componenet and [pass](https://github.com/ethereum-optimism/optimism/blob/71928829ca7ece48152159daa1d231eac2df03b3/op-node/rollup/derive/l1_retrieval.go#L57) it to the `DataSourceFactory` component, via `OpenData` function, similar to how [`SystemConfig.BatcherAddr`](https://github.com/ethereum-optimism/optimism/blob/71928829ca7ece48152159daa1d231eac2df03b3/op-service/eth/types.go#L382) is handled.
3. The `FetchingAttributesBuilder` component is updated to incorporate the canonical batch inbox into the `DepositTx`, via [`L1InfoDeposit`](https://github.com/ethereum-optimism/optimism/blob/71928829ca7ece48152159daa1d231eac2df03b3/op-node/rollup/derive/l1_block_info.go#L263). This modified `DepositTx` is subsequently used to obtain the `SystemConfig` during L2 chain reorganization.

During L2 reorganization, `op-node` knows the canonical batch inbox using the `SystemConfig` parameter of [`ResettableStage.Reset(context.Context, eth.L1BlockRef, eth.SystemConfig)`](https://github.com/ethereum-optimism/optimism/blob/71928829ca7ece48152159daa1d231eac2df03b3/op-node/rollup/derive/pipeline.go#L38) function, where `SystemConfig` is derived from the `DepositTx` of the corresponding L2 block.

### How `op-batcher` knows canonical batch inbox

Immediately before submitting a new batch, `op-batcher` fetches the current inbox address from L1 and submits to that address. After the transaction is successfully included in L1 at block `N`, `op-batcher` verifies that the inbox address hasn't changed at block `N`. If the address has changed, it resubmits the batch to the new address.

## Upgrade

Existing OP Stack instances need to upgrade the `SystemConfig` in order to use this feature.

## Reference Implementation

1. [example inbox contract for EthStorage](https://github.com/blockchaindevsh/es-op-batchinbox/blob/main/src/BatchInbox.sol)
2. [op-node & op-batcher changes](https://github.com/blockchaindevsh/optimism/compare/5137f3b74c6ebcac4f0f5a118b0f4909df03aec6...02e3b7248f1b590a2adf1f81488829760fa2ba03)

TODO: implement `Dynamic Updates` mentioned above.