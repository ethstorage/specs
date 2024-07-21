# L2 Blob Transaction

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [L2 Blob Transaction](#l2-blob-transaction)
  - [Motivation](#motivation)
  - [How It Works](#how-it-works)
  - [Reference Implementation](#reference-implementation)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Motivation

To make it seamless for dapps that's depending on blob transaction to migrate from L1 to L2, it's important that L2 also supports blob transaction. This will also make L3 more uniform since they can also use blob transaction for batch submitting, just like L2.

## How It Works

The changes include two parts: optimism and op-geth.

The optimism part mainly includes:
1. Add `L2BlobConfig` field to `rollup.Config` so that if configured, L2 blob tx will be supported and optionally the blob can be stored to a [DA provider](https://github.com/ethstorage/da-server) by sequencer.
   1. There're a bunch of workarounds to make L2 blob tx work for op stack:
      1. [`BuildBlocksValidator`](https://github.com/blockchaindevsh/optimism/blob/106fbfd5d45efa45d6580cf169a5e053e406b933/op-node/p2p/gossip.go#L250) is modified to support non-zero `BlobGasUsed` and `ExcessBlobGas`.
      2. [`spanBatchBlobTxData`](https://github.com/blockchaindevsh/optimism/blob/106fbfd5d45efa45d6580cf169a5e053e406b933/op-node/rollup/derive/span_batch_tx.go#L49) is added to support blob tx for span batch.
      3. [`CheckBlockHash`](https://github.com/blockchaindevsh/optimism/blob/106fbfd5d45efa45d6580cf169a5e053e406b933/op-service/eth/types.go#L218) is modified to support broadcasting blocks with blob tx.
      4. [`SimpleTxManager.craftTx`](https://github.com/blockchaindevsh/optimism/blob/106fbfd5d45efa45d6580cf169a5e053e406b933/op-service/txmgr/txmgr.go#L252) is modified to support blob tx.
      5. A `p2p.sync.onlyreqtostatic` flag is [added](https://github.com/ethereum-optimism/optimism/pull/11011) to support fetching blocks from static peers only.
2. For L2 blob tx, only blob hash is submitted to L1 by op-batcher, so a challenge mechanism for blob is planned.


The op-geth part mainly includes:
1. Add `EnableL2Blob` field to `OptimismConfig` so that if configured, L2 blob tx will be supported.
2. Add `blobs` field to `RollupCostData` and update [`newL1CostFuncEcotone`](https://github.com/blockchaindevsh/op-geth/blob/19971ab7ffd0b6879796daf8c05b3206e279f15d/core/types/rollup_cost.go#L191) so that a blob tx will have two extra fees: da fee and da proof fee.
3. Fix [`Transaction.RollupCostData`](https://github.com/blockchaindevsh/op-geth/blob/19971ab7ffd0b6879796daf8c05b3206e279f15d/core/types/transaction.go#L374) so that the result is consistent when called from worker and state processor.
4. [`commitBlobTransaction`](https://github.com/blockchaindevsh/op-geth/blob/19971ab7ffd0b6879796daf8c05b3206e279f15d/miner/worker.go#L806) is modified to support blob tx without blobs when deriving.
5. Port changes from upstream geth to support estimate gas for blob tx.

## Reference Implementation

1. [optimism repo changes](https://github.com/ethstorage/optimism/pull/23)
2. [op-geth repo changes](https://github.com/ethstorage/op-geth/pull/2)