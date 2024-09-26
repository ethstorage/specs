<!-- omit in toc -->
# L2 Blob Transaction

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Motivation](#motivation)
- [How It Works](#how-it-works)
- [Reference Implementation](#reference-implementation)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Motivation

The proposal aims to integrate BLOB transaction support into the OP Stack. BLOB transactions are gaining popularity on Layer 1 (L1), with applications including BLOB inscriptions, Rollups (as an optional feature), and EthStorage. However, none of the L2s (including OP Stack) supports BLOB transactions, resulting in high migration costs for these projects. To harvest the reduced gas costs of L2s with minimized migration costs of applications using BLOB transactions, it is of great value in OP Stack to support L2 BLOB transactions.

To further reduce the DA cost of L2 BLOB transactions of OP Stack, the proposal offers a hybrid DA solution, where the L2 BLOBs may use a different DA (e.g., DA committee or DA challenge) as that of L2 calldata (generally L1 DA).  This better serves the applications with demands on different data values with lower costs:
- For transactions with high-value data (e.g., swap in Uniswap), the users can use non-BLOB L2 transactions.
- For transactions with low-value data (e.g., NFT images/inscriptions), the users can use BLOB L2 transactions at a lower cost.

## How It Works

The proposed changes are implemented across two repositories: `optimism` and `op-geth`.

Changes in the `optimism` repository::
1. Addition of `L2BlobConfig` field to `rollup.Config` to enable L2 BLOB transaction support and optional BLOB storage to a Data Availability (DA) [provider]((https://github.com/ethstorage/da-server)) by the sequencer.
   - Several modifications to facilitate L2 BLOB transactions in OP Stack:
      - [`BuildBlocksValidator`](https://github.com/blockchaindevsh/optimism/blob/106fbfd5d45efa45d6580cf169a5e053e406b933/op-node/p2p/gossip.go#L250) updated to support non-zero `BlobGasUsed` and `ExcessBlobGas`.
      - Introduction of [`spanBatchBlobTxData`](https://github.com/blockchaindevsh/optimism/blob/106fbfd5d45efa45d6580cf169a5e053e406b933/op-node/rollup/derive/span_batch_tx.go#L49) for BLOB transaction support in span batch.
      - [`CheckBlockHash`](https://github.com/blockchaindevsh/optimism/blob/106fbfd5d45efa45d6580cf169a5e053e406b933/op-service/eth/types.go#L218) modified to enable broadcasting blocks with BLOB transactions.
      - [`SimpleTxManager.craftTx`](https://github.com/blockchaindevsh/optimism/blob/106fbfd5d45efa45d6580cf169a5e053e406b933/op-service/txmgr/txmgr.go#L252) updated to accommodate BLOB transactions.
      - [Addition]((https://github.com/ethereum-optimism/optimism/pull/11011)) of `p2p.sync.onlyreqtostatic` flag is to allow fetching blocks exclusively from static peers.
2. Implementation of a challenge mechanism for BLOB transactions is planned, as only BLOB hashes are submitted to L1 by op-batcher.


Changes in the `op-geth` repository:
1. Addition of `EnableL2Blob` field to `OptimismConfig` to enable L2 BLOB transaction support.
2. Inclusion of `blobs` field to `RollupCostData` and update of [`newL1CostFuncEcotone`](https://github.com/blockchaindevsh/op-geth/blob/19971ab7ffd0b6879796daf8c05b3206e279f15d/core/types/rollup_cost.go#L191) to introduce additional DA fees for BLOB transactions.
3. Correction of [`Transaction.RollupCostData`](https://github.com/blockchaindevsh/op-geth/blob/19971ab7ffd0b6879796daf8c05b3206e279f15d/core/types/transaction.go#L374) to ensure consistent results when called from worker and state processor.
4. Modification of [`commitBlobTransaction`](https://github.com/blockchaindevsh/op-geth/blob/19971ab7ffd0b6879796daf8c05b3206e279f15d/miner/worker.go#L806) to support BLOB transactions without blobs during derivation.
5. Integration of upstream geth changes to support gas estimation for BLOB transactions.

## Reference Implementation

1. [optimism repo changes](https://github.com/ethstorage/optimism/pull/23)
2. [op-geth repo changes](https://github.com/ethstorage/op-geth/pull/2)