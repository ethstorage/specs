# Inbox Contract

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

The integration process consists of four primary components:
1. Replacement of the [`BatchInboxAddress`](https://github.com/ethereum-optimism/optimism/blob/db107794c0b755bc38a8c62f11c49320c95c73db/op-chain-ops/genesis/config.go#L77) with an inbox contract: The existing `BatchInboxAddress`, which currently points to an Externally Owned Account (EOA), will be replaced by a smart contract. This new inbox contract will be responsible for verifying and enforcing batch submission conditions.
2. Modification of the op-node derivation process: The op-node will be updated to exclude failed batch transactions during the derivation process. This change ensures that only successfully executed batch transactions are processed and included in the derived state. 
3. Modification of the op-batcher submission process: The op-batcher will be updated to [call `recordFailedTx`](https://github.com/blockchaindevsh/optimism/blob/02e3b7248f1b590a2adf1f81488829760fa2ba03/op-batcher/batcher/driver.go#L537) for failed batch transactions. This modification ensures that the data contained in failed transactions will be resubmitted automatically.
   1. Most failures will be detected during the [`EstimateGas`](https://github.com/ethereum-optimism/optimism/blob/8f516faf42da416c02355f9981add3137a3db190/op-service/txmgr/txmgr.go#L266) call. However, under certain race conditions, failures may occur after the transaction has been included in a block.
4. To implement this feature as an optional setting, we introduced a `UseInboxContract` boolean field in both the `DeployConfig` and `rollup.Config` structures. When `UseInboxContract` is set to false, the system maintains its previous behavior, ensuring backward compatibility.

These modifications aim to enhance the security and efficiency of the batch submission and processing pipeline, allowing for more flexible and customizable conditions while maintaining the integrity of the derived state.


## Migration Process

A new function `setBatchInbox` will be introduced to the `SystemConfig` contract, enabling dynamic updates to the [`BatchInboxAddress`](https://github.com/ethereum-optimism/optimism/blob/8b1b67021fb77d7e33118b749679480532fce976/op-node/rollup/types.go#L120):

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

    bytes memory data = abi.encode(_batchInbox, _batchInbox.code.length > 0);
    emit ConfigUpdate(VERSION, UpdateType.BATCH_INBOX, data);
}

enum UpdateType {
    ...
    BATCH_INBOX
}
```

The `UseInboxContract` flag is automatically set to true for both the `op-node` and `op-batcher` components if `_batchInbox` corresponds to a valid contract address. If `_batchInbox` is not a valid contract address, the flag is set to false.

## Reference Implementation

1. [example inbox contract for EthStorage](https://github.com/blockchaindevsh/es-op-batchinbox/blob/main/src/BatchInbox.sol)
2. [op-node & op-batcher changes](https://github.com/blockchaindevsh/optimism/compare/5137f3b74c6ebcac4f0f5a118b0f4909df03aec6...02e3b7248f1b590a2adf1f81488829760fa2ba03)

TODO: implement `Migration Process` mentioned above.