---
tags:
  - substrate
keywords: [polkadot, substrate, hashing algorithm, storage, pallets]
description: Blake2 vs TwoX
updated: 2023-09-06
author: cenwadike
duration: 3h
level: intermediate
---

# Blake2 vs TwoX

It is a common practice to the hash input of storage items when implementing a
storage type with substrate. You may need to choose suitable hashing algorithms
to ensure data safety and maintain a well-optimized runtime by using an
appropriate algorithm for your specific use case to prevent a bottleneck
during state transition.

This guide will walk you through a common problem and pitfalls faced when using
a hashing algorithm on storage items that leverage FRAME hashing APIs and
provide an in-depth explanation of relevant concepts.

>Help us measure our progress and improve Substrate in Bits content by filling
out our living [feedback form](https://airtable.com/shr7CrrZ5zqlhWEUD).
It will only take 2 minutes of your time. Thank you!

## Reproducing errors

To reproduce the errors for this project you’ll have to set up your environment
and clone the repository of the node for this project.

## **Environment and project setup.**

To follow along with this tutorial, ensure that you have the Rust toolchain
installed.

- Visit the [substrate official documentation](https://docs.substrate.io/install/)
for the installation processes.

- Clone this project’s
[repository](https://github.com/cenwadike/Blake2_128Concat-vs-Twox64Concat).

```rust
git clone https://github.com/cenwadike/Blake2_128Concat-vs-Twox64Concat
```

- Navigate into the project’s directory.

```bash
cd Blake2_128Concat-vs-Twox64Concat
```

Navigate to faulty implementation.

```bash
git checkout a5a1bcc
```

- Run the command below to build the pallet.

```bash
cargo build --release
```

_**The error is line [42](https://github.com/cenwadike/Blake2_128Concat-vs-Twox64Concat/commit/a5a1bcc573b0bb2c4ff93bc440fb8d38780c49f8#diff-b1a35a68f14e696205874893c07fd24fdb88882b47c23cc0e0c80a30c7d53759R42:~:text=_%2C-,Twox64Concat%2C,-//%20NOTE%3A%20Blake2_128Concat%20should) of lib.rs.**_

You would notice that no error message appeared in the terminal. This is
because this error is silent, and only becomes apparent when abused by a
malicious user.

## Solving the error

Before delving into the details of this error, let us have a look at the
storage map that the error originates from.

```rust
  #[pallet::storage]
  pub(super) type ArchiveStore<T: Config> = StorageMap<
    _,
    Twox64Concat, // NOTE: Blake2_128Concat should be used here
    T::Hash,
    BookSummary<T::AccountId, T::BlockNumber>,
    OptionQuery,
  >;
```

`ArchiveStore` is a storage map that uses key-value pairs to store summaries
about books and assign a unique key to each book summary.
A key (of type `Hash`) can be used to randomly lookup the summary of a book if
it exists in `ArchiveStore`.

To enable storage into runtime state trie and on-chain data verification,
substrate FRAME provides different hashing algorithms(aka hashers), to
retrieve the data from the state trie.

As a general principle, `Blake2_128Concat` should be your default hasher
(we will get back to this shortly).

We can apply a more appropriate hasher by changing `Twox64Concat` to
`Blake2_128Concat` like so:

```rust
  #[pallet::storage]
  pub(super) type ArchiveStore<T: Config> = StorageMap<
    _,
    Blake2_128Concat, // NOTE: Blake2_128Concat should be used here
    T::Hash,
    BookSummary<T::AccountId, T::BlockNumber>,
    OptionQuery,
  >;
```

- Recompile the pallet

```bash
cargo build --release
```

We have been able to use a `Blake2_128Concat`, which is a more appropriate
hasher for `ArchiveStore`.
`Blake2_128Concat` produces a hash from the key of each book summary and
concatenates the key to the end of the hash.

## Going in-depth

### Data Abstractions

Substrate `FRAME` offers several storage abstractions when handling data in
substrate. This is mainly due to the complexity of data handling in the context
of a blockchain; ensuring efficient read and write while ensuring flexibility
for data verification and consensus.

At its core, substrate uses a simple key-value data store implemented as a
database-backed, modified Merkle tree. This allows all higher-level storage
abstractions to be built on top of this simple key-value store.

This key-value database stores different instances of
Base-16 Modified Merkle Patricia tree ("trie"), including a single `state trie`
and `trie` of other forks of the `state`.

A child trie can be generated from the `state trie` for several purposes for
example event verification in cross-chain verification. It is important to note
the child trie is not separately stored on the key-value database and gets
updated as the state (parent) trie evolves.

As mentioned above, substrate `FRAME` provides several higher-level storage
abstractions including `StorageMap`.

`StorageMap` also called _single-key storage map_ is a FRAME storage
abstraction that is ideal for managing sets of items whose elements will be
accessed randomly, using a key-to-value mapping to perform random lookups.

`StorageMap` is defined like so:

```rust
pub struct StorageMap<
 Prefix,
 Hasher,
 Key,
 Value,
 QueryKind = OptionQuery,
 OnEmpty = GetDefault,
 MaxValues = GetDefault,
>(core::marker::PhantomData<(Prefix, Hasher, Key, Value, QueryKind, OnEmpty, MaxValues)>);
```

From the code snippet above, we can see that StorageMap allows us to specify a
hasher (among other parameters).

### Blake2_128Concat

`Blake2_128Concat` is a hashing algorithm that uses
[blake2b](https://www.blake2.net/) cryptographic hash function to generate a
hash of a key and appends the key to the hash generated.

The pseudocode below demonstrates how `Blake2_128Concat` could work.

```rust
let key = _key;
let storage_key_hash;

function Blake2_128Concat(key) -> hex {
  key_hash_as_hex = blake2::hash(key);

  storage_key_hash = key_hash_as_hex.concatenate(key);

  return storage_key_hash;
}
```

By appending the key to key_hash, anyone who knows the key can verify that the
hash was generated from that key. Because blake2 generates a cryptographically
secured hash, ie. you can not guess the preimage of a hash, only those who know
a key can verify that a hash was created from it.

Because only key knowers (in some cases owners) can verify a hash of key and
get access to data mapped to that key. This eliminates an attack vector where
a malicious user can distort data or bombard the runtime with data updates,
which may make the data read more time-consuming due to `imbalance trie`.

### Twox_64Concat

`Twox_64Concat` is a hashing algorithm that uses
[xxhash](https://create.stephan-brumme.com/xxhash/) non-cryptographic hash
function to generate a hash of a key and appends the key to the hash generated.

`Twox_64Concat` work similarly to `Blake2_128Concat`, concatenating a key to
its hash.

Because `Twox64Concat` is not cryptographically secure, it is not ideal where
keys are supplied by users or limited access to data is desired.

However, it is important to note that `Twox64Concat` is ideal for use cases
where speed is desirable over data access limitation at the level of the data
structure (You may need to implement a separate authorization). `Twox64Concat`
is also ideal where the key is sufficiently cryptographically secured for the use case.

As we have seen, the choice of hasher is one that is made on a case-by-case
basis and requires a deep understanding of the system you want to  build to
gauge between speed and cryptographic difficulty.

## Summary

This guide provides an in-depth exploration into the role of FRAME storage
abstraction and examined `Blake2_128Concat` and `Twox64Concat` hashing
algorithms when storing data in `StorageMap` on a substrate runtime.

`Blake2_128Concat` generates a cryptographically secured hash of keys that can
be verified and used to access data on a substrate runtime state.

`Twox64Concat` generates fast non-cryptographic hash of keys, that are ideal
for low latency use cases.

Using an error-led approach, we highlighted how to decide on which hashing
algorithm to use for different use cases.

We developed an understanding of substrate data abstractions and saw how
high-level abstractions like `StorageMap` store data on-chain.

We also had a look at how data is stored as a Patricia tree in a key-value
database.

For more resources on the concepts discussed in this article, check out:

- [runtime storage](https://docs.substrate.io/build/runtime-storage/#single-key-storage-maps)
- [State transitions and storage](https://docs.substrate.io/learn/state-transitions-and-storage/)
- [frame_support blake2_128Concat implementation](https://paritytech.github.io/substrate/master/frame_support/struct.Blake2_128Concat.html)
- [frame_support StorageMap](https://paritytech.github.io/substrate/master/src/frame_support/storage/types/map.rs.html#47-55)
- [ShawnT's Transparent Keys in Substrate](https://www.shawntabrizi.com/substrate/transparent-keys-in-substrate/)

> We’re inviting you to
[fill out our living feedback](https://airtable.com/shr7CrrZ5zqlhWEUD) form to
help us measure our progress and improve Substrate in Bits content. It will
only take 2 minutes of your time. Thank you!
