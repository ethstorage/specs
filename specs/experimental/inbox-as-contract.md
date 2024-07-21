# Inbox as Contract

## Motivation

Currently the batch inbox is an EOA, this spec aims to allow the batch inbox to be a contract, to enable customized batch submission conditions like:
1. The batch tx must be signed by a quorom of shared sequencers.
2. The batch tx must pay DA fee to EthStorage to enable permanent and decentralized storage.

## How It Works

The integration mainly includes two parts:
1. The [`BatchInboxAddress`](https://github.com/ethereum-optimism/optimism/blob/db107794c0b755bc38a8c62f11c49320c95c73db/op-chain-ops/genesis/config.go#L77) is replaced by an inbox contract to check submission conditions.
2. When deriving, op-node should exclude failed batches.



## Reference Implementation

1. [example inbox contract for EthStorage](https://github.com/blockchaindevsh/es-op-batchinbox/blob/main/src/BatchInbox.sol)
2. [op-node derive changes](https://github.com/ethstorage/optimism/pull/22)