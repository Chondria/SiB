---
tags:
  - substrate
keywords: [polkadot, substrate, storage, tries, kvdb]
description: Deep Dive into Substrate Storage - Part 2
updated: 2023-12-16
author: cenwadike
duration: 3h
level: advanced
---

# Deep Dive into Substrate Storage - Part 2

Data on the blockchain is eventually stored in various forms at different
points in its lifecycle. From transactional data temporarily held in the
transaction pool, to blocks that are gossiped through the network, and
eventually stored by nodes, all data on the blockchain is eventually stored
in a database. This ensures data durability and isolation.

This is the second part of the series on Substrate storage. We will have an
overview of the data lifecycle within the scope of Substrate (from
transaction data to storage in a key-value database), develop an understanding
of how the Substrate database (backend) works, and how data read and write is
optimized by Substrate. We will also explore code implementations to further
buttress understanding of relevant concepts.

>Help us measure our progress and improve Substrate in Bits content by filling
out our living [feedback form](https://airtable.com/shr7CrrZ5zqlhWEUD).
It will only take 2 minutes of your time. Thank you!

## General overview of data storage in Substrate

It is important to have a good understanding of the basics of data storage in
the context of Substrate and by extension blockchains. The previous part of
this series provides in-depth descriptions of these basic concepts, however, we
will take a brief overview of the basics.

In a nutshell, Substrate provides **storage items** that enable developers to
create custom **data types** easily. These storage items and the subsequent
data types created from them are built on top of a storage abstraction called
**Trie**. Substrate uses **Patricia Trie** to package transactions into blocks,
abstract the key-value database, and provide efficient data handling within
nodes and gossiping with other nodes.

> Check out part of this series [here](./deep-dive-into-storage-part-1.md) to
have a deeper understanding of the basics.

## Data Lifecycle - From Transaction Data to Key-Value DB

### Transaction Data

The *lifecycle* of data on the blockchain begins from a `transaction`.
Transaction also called `Extrinsic` frequently contains data relevant to a
runtime function call; for example the destination address for a token
transfer. 

You can conceptualize an extrinsic with its data like so:

```rust
type Extrinsic {
    pub_key: AccountId,             // senders pub-key
    method: RuntimeCall,            // method name
    param: Option<ParamType>        // method parameters
    signature: Option<Signature>,   // senders signature
    data: SignedExtra,              // extra data
}
```

It is also important to note that before the completion of transaction
execution, transaction data are stored in the **transactional storage layer**. The
transactional storage layer temporarily stores transaction data and changes to the
underlying state change in memory. This ensures transaction atomicity,
preventing failed transaction state from being persisted in the blockchain
state and the state of only successfully executed transactions that are persisted in
the blockchain state.

### Blocks

`Block` is an important data structure in blockchains, they form the heartbeat
of a blockchain. They provide an efficient medium for processing, validating,
and storing transactions. We have provided a detailed description of blocks and
transactions in a previous [guide](./from-transaction-to-block-part-1.md).

Blocks can be conceptually visualized like so:

```rust
type Block {
    inherent: u8,
    block_header: Hash,
    transactions: Vec<ValidTransactions>,
}
```

Blocks undergo complex processes to ensure their validity following the consensus
mechanism of the blockchain before they eventually get stored by full nodes. It
is important that although data on the blockchain is transported and validated
as blocks, the `Block` data structure is **not** the storage form of data. Data
on the blockchain is stored in the key-value database, and the `Block` is only a data
abstraction for efficient data processing.

### KVDB

Substrate provides a key-value database (KVDB) which eventually records valid
data on the blockchain. This KVDB forms the basis of storage on a blockchain,
providing a solid foundation for different abstractions including `Tries` and
`Blocks`.

By default, Substrate uses [`Rocks DB`](https://rocksdb.org/), for persisting
data. Alternatively, you can port any key-value database that caters to your
use case, or explore using [`Parity DB`](https://github.com/paritytech/parity-db)

In the subsequent section, we will dive into the blockchain backend that
handles data storage.

## Deep dive into KVDB - Data Backend and Outer Client

TODO!

## Summary

In this guide, we developed an understanding of how data is eventually stored
in Substrate.

We developed an understanding of the following:

- What data lifecycle entails in Substrate.
- How the outer client manages the KVDB for data storage.

To learn more about Substrate storage items and Merkle trie, check out these
resources:

- [Storage Items](https://docs.substrate.io/build/runtime-storage/#transactional-storage)
- [Shawn T's Storage Deep Dive](https://www.shawntabrizi.com/assets/presentations/substrate-storage-deep-dive.pdf)

>Help us measure our progress and improve Substrate in Bits content by filling
out our living [feedback form](https://airtable.com/shr7CrrZ5zqlhWEUD).
Thank you!
