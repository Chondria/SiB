---
tags:
  - substrate
keywords: [polkadot, substrate, pallets, hooks]
description: Working with substrate pallet hooks.
updated: 2023-07-24
author: cenwadike
duration: 2h
level: intermediate
---

# Working with substrate pallet hooks
Substrate pallet hooks are a powerful set of tools substrate provides that facilitates automatic runtime execution when some condition is met.

Substrate pallet hooks enable runtime engineers to implement automatic runtime execution of arbitrary processes under deterministic conditions and with a verifiable guarantee of runtime safety.

In this guide, we will break down important concepts about substrate pallet hooks and dive into how hooks can be implemented and executed on a runtime.

>Help us measure our progress and improve Substrate in Bits content by filling out our living [feedback form](https://airtable.com/shr7CrrZ5zqlhWEUD). Thank you!


## Reproducing setup

### Environment and project setup

To follow along with this tutorial, ensure that you have the Rust toolchain installed.

- Visit the [substrate official documentation](https://docs.substrate.io/install/) page for the installation processes.

- Clone the project [repository](https://github.com/cenwadike/double-auction).

```rust
git clone https://github.com/cenwadike/double-auction
```

- Navigate into the project’s directory.

```
cd double-auction
```

- Run the command below to build the pallet.

```
cargo build --release
```


## Getting some context

The setup below is a substrate pallet that implements [double-auction](https://en.wikipedia.org/wiki/Double_auction) for electrical energy.

The seller gets matched (and their electricity is sold to the highest bidder) when the auction period is over.

Auctions to be executed are stored in *`AuctionsExecutionQueue`*. 
When an auction period is over, it is taken from the *`AuctionsExecutionQueue`* and matched to the highest bidder.

Using *`on_finalize`* hook, the runtime checks if any auction is over, and executes the auction by calling *`on_auction_ended`*.

*`on_auction_ended`* is executed after all runtime extrinsic have been executed. 

*`on_auction_ended`* is also executed within the constraints of the `Weight` assigned to it in *`on_initialize`*


## Understanding substrate pallet hooks

Substrate provides a highly extensible interface that provides a comprehensive set of runtime states to which arbitrary execution could be anchored.

This interface is implemented like so:
```rust
  pub trait Hooks<BlockNumber> {
    // called at the very beginning of block execution. 
    fn on_initialize(_n: BlockNumber) -> Weight { ... }

    // called at the very end of block execution. 
    // requires on_initialize
    fn on_finalize(_n: BlockNumber) { ... }

    // consume a block’s idle time.
    // run when the block is being finalized, before on_finalize
    fn on_idle(_n: BlockNumber, _remaining_weight: Weight) -> Weight { ... }

    // hooks for runtime upgrades
    fn on_runtime_upgrade() -> Weight { ... }    
    fn pre_upgrade() -> Result<Vec<u8>, TryRuntimeError> { ... }
    fn post_upgrade(_state: Vec<u8>) -> Result<(), TryRuntimeError> { ... }

    // hook sanity checks of a pallet, per block.
    fn try_state(_n: BlockNumber) -> Result<(), TryRuntimeError> { ... }

    // useful for anchoring off-chain workers.
    fn offchain_worker(_n: BlockNumber) { ... }

    // to check the integrity of this pallet’s configuration
    fn integrity_test() { ... }
  }
```

For full Trait documentation, check the [docs](https://paritytech.github.io/substrate/master/frame_support/traits/trait.Hooks.html)

You can couple substrate `Hooks` trait into your pallet and implement the *`on_initialize`* and *`on_finalize`* like so:

```rust
  #[pallet::hooks]
  impl<T: Config> Hooks<T::BlockNumber> for Pallet<T> {
    fn on_initialize(_now: T::BlockNumber) -> Weight {
      // Return weight required for on_finalize dispatch
      T::WeightInfo::on_finalize(AuctionsExecutionQueue::<T>::iter_prefix(now).count() as u32)
    }

    fn on_finalize(now: T::BlockNumber) {
        // Get auction ready for execution
        for (auction_id, _) in AuctionsExecutionQueue::<T>::drain_prefix(now) {
            if let Some(auction) = Auctions::<T>::take(auction_id) {
                // handle auction execution
                Self::on_auction_ended(auction.auction_id);
            }
        }
    }
  }
```

To view the full implementation details of substrate *`Hooks`* trait, check the [docs](https://paritytech.github.io/substrate/master/src/frame_support/traits/hooks.rs.html)


## Going in-depth

This is merely an umbrella trait for traits that define each method in `Hooks` trait. 
You can have a mental picture of this like so:

```rust
mod Hooks {
  pub trait OnInitialize<BlockNumber> {
    // Provided method
    fn on_initialize(_n: BlockNumber) -> Weight { ... }
  }

  pub trait OnFinalize<BlockNumber> {
    // Provided method
    fn on_finalize(_n: BlockNumber) { ... }
  }
  
  // -------- snip -----------
}
```

When we implement a pallet hook as shown below, we are leveraging substrate to hide the complexity of implementing multiple traits for our pallet:

```rust
  #[pallet::hooks]
  impl<T: Config> Hooks<T::BlockNumber> for Pallet<T> {
      // -------- snip -----------
  }
```

You could also use only the traits relevant to your pallet like so:
```rust
  #[pallet::hooks]
  impl<T: Config> OnInitialize<T::BlockNumber> for Pallet<T> {
      // -------- snip -----------
  }
```

At execution, substrate runtime orchestrator implemented as *`frame_executive`* correctly dispatch a pallet's hook and execute any code.

`frame_executive` works in conjunction with the FRAME System module and serves as the main entry point for certain runtime actions including:
- Check transaction validity.
- Initialize a block.
- Apply extrinsics.
- Execute a block.
- Finalize a block.
- Start an off-chain worker.

In a nutshell, it is the `frame_executive` that oversees the execution of hooks defined in our pallets.


## Summary
We used a double auction pallet to demonstrate how to couple substrate hooks to a pallet. We explored substrate pallet hooks and developed an understanding of:
- different methods exposed by substrate `Hooks` trait.
- how hooks are executed by [frame-executive](https://paritytech.github.io/substrate/master/frame_executive/index.html).

Substrate pallet hooks are a powerful and useful set of tools that can be used to add dynamism to runtime execution. `Hooks` are highly extensible facilitating different use cases.

To learn more about testing in substrate, check out these resources:
- [Substrate Hooks Trait implementation](https://docs.substrate.io/reference/how-to-guides/weights/add-benchmarks/)
- [Substrate Hooks Trait documentation](https://docs.substrate.io/test/benchmark/)
- [Frame executive](https://paritytech.github.io/substrate/master/frame_executive/index.html)
