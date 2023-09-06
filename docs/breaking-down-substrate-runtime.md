---
tags:
  - substrate
keywords: [polkadot, substrate, runtime, architecture, implementation]
description: Breaking down substrate runtime
updated: 2023-09-06
author: cenwadike
duration: 3h
level: advanced
---

# Breaking down substrate runtime

At the heart of every blockchain lies a contained environment called the
runtime that orchestrates the core functionalities of a blockchain. These
functionalities enable the blockchain to accurately execute the logic of state
transition functions including consensus, transaction execution, and on-chain
state definition and validation.

Substrate provides a unique approach to runtime design and implementation.
Substrate unique approach and highly modularized approach to runtime design and
development is composed of highly specialized
[FRAME](https://docs.substrate.io/reference/glossary/#frame) modules. FRAME is
a highly flexible and composable building block that handles every runtime
process, providing an excellent framework for building powerful runtimes.

In this guide, we will dive into the substrate runtime. We will help you
demystify substrate runtime architecture, and highlight the implementation of
essential FRAME modules that are indispensable when building a substrate
runtime.

> Help us measure our progress and improve Substrate in Bits content by
filling out our living [feedback form](https://airtable.com/shr7CrrZ5zqlhWEUD).
Thank you!

## Reproducing setup

### Project setup

To follow along, ensure you have the Rust toolchain installed.

- Visit the
[substrate official documentation](https://docs.substrate.io/install/)
for the installation processes.

- Clone the project
[repository](https://github.com/cenwadike/double-auction-node).

```rust
git clone https://github.com/cenwadike/double-auction-node
```

- Navigate into the project’s directory.

```bash
cd double-auction-node
```

- Run the command below to build the pallet.

```bash
cargo build --release
```

## Getting some context

The setup above is a substrate node that implements
[double-auction](https://en.wikipedia.org/wiki/Double_auction) for electrical
energy.

When the auction period ends, the seller gets matched
(and their electricity is sold to the highest bidder).

When an auction ends, it is taken from the _`AuctionsExecutionQueue`_ and
matched to the highest bidder.

Use the setup above as a reference to follow along if you get stuck.

## Understanding substrate runtime architecture

Substrate runtime is a major component of substrate node
[architecture](https://docs.substrate.io/learn/architecture/).
The node itself is composed of:

- A _runtime_ that contains all of the business logic for executing the state
transition function of the blockchain.

- A _core client_ with _outer node services_ that handles network activity such
as peer discovery, responding to RPC calls, managing transaction requests, and
reaching consensus with peers. We will break down how these features are
achieved in subsequent guides.

The outer node services communicate with runtime using APIs exposed by the
runtime. The runtime API is flexible in design and must contain an interface
for _`Core`_ which exposes block execution features and _`Metadata`_ of the
runtime. We will look at these interfaces
[later](#understanding-substrate-runtime-implementation).

In addition to `FRAME`, substrate provides primitives that the runtime must
implement. These substrate primitives provide a standard for ubiquitous
components used throughout the node. These core primitives can be defined
however you want to fit your blockchain need, bearing in mind the security
considerations and compatibility with different parts of substrate.

To fit your blockchain into what substrate already provides, the following core
primitives must be defined in your runtime as described
[here](https://docs.substrate.io/learn/runtime-development/#core-primitives).

## Understanding substrate runtime implementation

From the previous [section](#understanding-substrate-runtime-architecture), we
were able to develop a mental picture of the composition of the substrate
runtime and how it exposes APIs to communicate with the outer node. Here, we
will take a hands-on approach to understand the implementation details of
substrate runtime.

We will use the code found in `runtime/src/lib.rs` as a starting point to break
down some of the implementation details of a substrate runtime.

The runtime implemented in `runtime/src/lib.rs` can be roughly divided into the
following sections:

- (Primitive) type declaration

- Runtime configuration

- Runtime benchmarking

- Runtime API implementation

It is also important to note that substrate runtime leverages web assembly
(WASM) as the execution environment to ensure multiplatform compatibility.
Hence, it is implemented without using the Rust standard library.

This configuration is set at the top of `runtime/src/lib.rs` like so:

```rust
#![cfg_attr(not(feature = "std"), no_std)]
```

The runtime can also tag processes to be executed in the execution environment
of the host machine.

### Primitive type declaration

Substrate runtime implementation is highly modular and extensively relies on
multiple crates for `type` definition, `traits`, and other complex data
components.

Important crates worth mentioning include `sp_runtime`,
`sp_api::impl_runtime_apis`, `frame_support`, and `frame_system`. They offer
helpers components that are used to implement FRAME pallets on the runtime.

You can also import relevant FRAME pallets you want to use for your runtime
and bring in the custom pallets relevant to your blockchain like so:

```rust

// ------------------snip------------------

use sp_api::impl_runtime_apis; // <--- expose runtime APIs
use sp_consensus_aura::sr25519::AuthorityId as AuraId; // <--- block authoring
use sp_core::{crypto::KeyTypeId, OpaqueMetadata};
use sp_runtime::{
create_runtime_str, generic, impl_opaque_keys,
traits::{
  AccountIdLookup, BlakeTwo256, Block as BlockT, IdentifyAccount, NumberFor, One, Verify,
},
transaction_validity::{TransactionSource, TransactionValidity},
ApplyExtrinsicResult, MultiSignature,
};
use sp_std::prelude::*;
#[cfg(feature = "std")] // <--- execute outside wasm runtime
use sp_version::NativeVersion;
use sp_version::RuntimeVersion;

// A few exports that help ease life for downstream crates.
pub use frame_support::{
construct_runtime, parameter_types,
traits::{
  ConstBool, ConstU128, ConstU32, ConstU64, ConstU8, KeyOwnerProofSystem, Randomness,
  StorageInfo,
},
weights::{
  constants::{
  BlockExecutionWeight, ExtrinsicBaseWeight, RocksDbWeight, WEIGHT_REF_TIME_PER_SECOND,
  },
  IdentityFee, Weight,
},
StorageValue,
};
pub use frame_system::Call as SystemCall;
pub use pallet_balances::Call as BalancesCall; // <--- for fungible token managment

// ------------------snip------------------

pub use pallet_template;  // <--- custom pallet 

```

You can declare the runtime primitive types like so:

```rust

// ------------------snip------------------

/// An index to a block.
pub type BlockNumber = u32;

/// Alias to 512-bit hash when used in the context of a transaction signature on the chain.
pub type Signature = MultiSignature;

/// Some way of identifying an account on the chain. We intentionally make it equivalent
/// to the public key of our transaction signing scheme.
pub type AccountId = <<Signature as Verify>::Signer as IdentifyAccount>::AccountId;

/// Balance of an account.
pub type Balance = u128;

/// Index of a transaction in the chain.
pub type Nonce = u32;

/// A hash of some data used by the chain.
pub type Hash = sp_core::H256;

// ------------------snip------------------

```

Other primitive types that may be relevant to your blockchain can be added as
shown above. Ensure that that they are compatible with FRAME (if you intend to
use FRAME pallets), and the outer node.

The runtime also declares some important constants including the block time and
constants used in different configurations.

### Runtime configuration

Similar to other software systems, the configuration of different Pallets used
to compose the runtime and that of the runtime itself is defined in this
section.

The runtime configuration gives a summary of all types exposed by the runtime.
Some of these types are simple rust primitive types like Nonce, while others
are complex types.

The runtime configuration is defined like so:

```rust

// ------------------snip------------------

// Configure FRAME pallets to include in runtime.

impl frame_system::Config for Runtime {
 /// The basic call filter to use in dispatchable.
 type BaseCallFilter = frame_support::traits::Everything;
 /// The block type for the runtime.
 type Block = Block;
 /// Block & extrinsics weights: base values and limits.
 type BlockWeights = BlockWeights;
 /// The maximum length of a block (in bytes).
 type BlockLength = BlockLength;
 /// The identifier used to distinguish between accounts.
 type AccountId = AccountId;
 /// The aggregated dispatch type that is available for extrinsics.
 type RuntimeCall = RuntimeCall;
 /// The lookup mechanism to get account ID from whatever is passed in dispatchers.
 type Lookup = AccountIdLookup<AccountId, ()>;
 /// The type for storing how many extrinsics an account has signed.
 type Nonce = Nonce;
 /// The type for hashing blocks and tries.
 type Hash = Hash;
 /// The hashing algorithm used.
 type Hashing = BlakeTwo256;
 /// The ubiquitous event type.
 type RuntimeEvent = RuntimeEvent;
 /// The ubiquitous origin type.
 type RuntimeOrigin = RuntimeOrigin;
 /// Maximum number of block number to block hash mappings to keep (oldest pruned first).
 type BlockHashCount = BlockHashCount;
 /// The weight of database operations that the runtime can invoke.
 type DbWeight = RocksDbWeight;
 /// Version of the runtime.
 type Version = Version;
 /// Converts a module to the index of the module in `construct_runtime!`.
 ///
 /// This type is being generated by `construct_runtime!`.
 type PalletInfo = PalletInfo;
 /// What to do if a new account is created.
 type OnNewAccount = ();
 /// What to do if an account is fully reaped from the system.
 type OnKilledAccount = ();
 /// The data to be stored in an account.
 type AccountData = pallet_balances::AccountData<Balance>;
 /// Weight information for the extrinsics of this pallet.
 type SystemWeightInfo = ();
 /// This is used as an identifier of the chain. 42 is the generic substrate prefix.
 type SS58Prefix = SS58Prefix;
 /// The set code logic, just the default since we're not a parachain.
 type OnSetCode = ();
 type MaxConsumers = frame_support::traits::ConstU32<16>;
}

// ------------------snip------------------

```

The runtime configuration is extended for each pallet to provide a simple
approach for using pallet-specific types on runtime only when needed.

An example configuration for a pallet is shown like so:

```rust

// ------------------snip------------------

impl pallet_template::Config for Runtime {
 type RuntimeEvent = RuntimeEvent;
 // type WeightInfo = pallet_template::weights::SubstrateWeight<Runtime>;
 type AuctionId = u64;
}

// ------------------snip------------------

```

### Runtime benchmarking

To provide verifiable certainty that a runtime process will be executed within
a reasonable amount of time (for example the block time), runtime benchmarks
are added to enable testing Pallet extrinsics.

The runtime defines a module that handles all benchmarking-related processes
like so:

```rust

// ------------------snip------------------

#[cfg(feature = "runtime-benchmarks")]
#[macro_use]
extern crate frame_benchmarking;

#[cfg(feature = "runtime-benchmarks")]
mod benches {
 define_benchmarks!(
  [frame_benchmarking, BaselineBench::<Runtime>]
  [frame_system, SystemBench::<Runtime>]
  [pallet_balances, Balances]
  [pallet_timestamp, Timestamp]
  [pallet_sudo, Sudo]
  [pallet_template, TemplateModule]
 );
}

// ------------------snip------------------

```

Runtime benchmarking is essential to provide security guarantees against DOS
attacks on your blockchain.

To learn how to benchmark your substrate runtime, check out this [guide](./Benchmarking-substrate-pallet.md).

### Runtime API implementation

The runtime API implementation composes the bulk of `runtime/src/lib.rs`.
It implements and exposes the functionalities of the runtime.

These functionalities are involved in transaction validation, metering and
execution, block creation, validation and finality, session key handling,
handling runtime benchmarking, and upgrades.

The _*Core*_, _*Metadata*_, and _*BlockBuilder*_ APIs were implemented like so:

```rust

// ------------------snip------------------

impl_runtime_apis! {
  /// block initialization and execution 
  impl sp_api::Core<Block> for Runtime {
    fn version() -> RuntimeVersion {
    VERSION
    }

    fn execute_block(block: Block) {
    Executive::execute_block(block);
    }

    fn initialize_block(header: &<Block as BlockT>::Header) {
    Executive::initialize_block(header)
    }
  }

  /// expose all runtime metadata
  impl sp_api::Metadata<Block> for Runtime {
    fn metadata() -> OpaqueMetadata {
    OpaqueMetadata::new(Runtime::metadata().into())
    }

    fn metadata_at_version(version: u32) -> Option<OpaqueMetadata> {
    Runtime::metadata_at_version(version)
    }

    fn metadata_versions() -> sp_std::vec::Vec<u32> {
    Runtime::metadata_versions()
    }
  }

  impl sp_block_builder::BlockBuilder<Block> for Runtime {
    fn apply_extrinsic(extrinsic: <Block as BlockT>::Extrinsic) -> ApplyExtrinsicResult {
    Executive::apply_extrinsic(extrinsic)
    }

    fn finalize_block() -> <Block as BlockT>::Header {
    Executive::finalize_block()
    }

    fn inherent_extrinsics(data: sp_inherents::InherentData) -> Vec<<Block as BlockT>::Extrinsic> {
    data.create_extrinsics()
    }

    fn check_inherents(
    block: Block,
    data: sp_inherents::InherentData,
    ) -> sp_inherents::CheckInherentsResult {
    data.check_extrinsics(&block)
    }
 }

```

Similarly, any runtime API can be implemented for a substrate runtime.  

## Summary

In a nutshell, substrate runtime is a contained environment composed of
multiple FRAME and non-FRAME crates and primitive types. We were able to
appreciate how we can implement and expose functionalities to the outer node.

We developed an understanding of:

- high-level architecture of substrate runtime.
- components of substrate runtime.
- primitive types of substrate runtime.

To learn more about substrate runtime, check out these resources:

- [Substrate Architecture](https://docs.substrate.io/learn/architecture/)
- [Substrate Runtime](https://docs.substrate.io/learn/runtime-development/)
- [impl_runtime_api](https://paritytech.github.io/substrate/master/sp_api/macro.impl_runtime_apis.html)
- [Runtime Execution](https://learnblockchain.cn/docs/substrate/docs/knowledgebase/runtime/execution/)(out-dated code)

We’re inviting you to fill our living [feedback form](https://airtable.com/shr7CrrZ5zqlhWEUD)
to help us measure our progress and improve Substrate in Bits content.
It will only take 2 minutes of your time. Thank you!
