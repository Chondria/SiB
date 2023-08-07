---
tags:
  - substrate
keywords: [polkadot, substrate, pallets, genesis, config]
description: Deconstructing substrate pallet genesis config.
updated: 2023-08-07
author: cenwadike
duration: 3h
level: intermediate
---


# Deconstructing substrate pallet genesis config
Substrate provides a smooth interface that allows blockchain runtime developers to define the configuration of pallet storage at the genesis block of the runtime or a pallet instantiation.

Substrate uses `genesis_config` and `genesis_build` macros in combination with the `GenesisBuild` trait to abstract complex APIs and processes involved in the initialization of a pallet's storage. 

In this guide, we will dive deep into the internals of substrate pallet storage configuration using a trivial example implemented on a double auction pallet.

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

Using substrate _genesis_config_ we initialize *`AuctionIndex`*. 

`AuctionIndex` is an incremental counter that assigns a unique id to an *`Auction`*.


## Adding pallet genesis config

To add the genesis configuration of a pallet storage item, we need to follow five steps:
- Define the storage items.
- Add storage item to `GenesisConfig`.
- Define default values of storage items.
- Implement `GenesisBuild` for `GenesisConfig`.
- Define concrete values for storage items.

_NB: The code used in this guide prioritizes readability over conciseness. Feel free to use more concise code where you see fit._ 

### Defining storage item

We used substrate `StorageValue` to define `AuctionIndex` as an item like so:

```rust
  #[pallet::storage]
  #[pallet::getter(fn auctions_index)]
  pub(super) type AuctionIndex<T: Config> = StorageValue<_, u64>;
```

### Adding the storage item to `GenesisConfig`

Substrate defines a convention to simplify the definition of a pallet genesis configuration to ease integration with other parts of substrate including substrate runtime and outer node.

We can define the genesis configuration as a struct like so:

```rust
  // define pallet's genesis configuration
  #[pallet::genesis_config]
  pub struct GenesisConfig {
      pub auction_index: u64,
  }
```

We could also define genesis config as an `enum`, however for this example, we will stick to using a `struct`.

### Defining the default value for a storage item

Defining a default value allows us to defer hard-coding initial values on a storage items. The assignment of the initial value is deferred to the runtime that implements our pallet genesis configuration.

The default value for `AuctionIndex` is defined like so:

```rust 
  // assign default value for storage items
  #[cfg(feature = "std")]
  impl Default for GenesisConfig {
      fn default() -> Self {
          Self {
              auction_index: Default::default(),
          }
      }
  }
```

This uses Rust's in-built default value for `u64`.

We could also define custom default values for complex types including substrate generic types. A link to a substrate doc describing how to use generic types will is added at the end of this guide.

### Implementing `GenesisBuild` for `GenesisConfig`

We can now implement the `GenesisBuild` trait on our `GenesisConfig` to expose a build function that is leveraged by the runtime to initialize the pallet storage items.

The code below implements `GenesisBuild` for `GenesisConfig` and defines the build function where the initial storage is passed into `AuctionIndex`.

```rust
  #[pallet::genesis_build]
  impl<T: Config> GenesisBuild<T> for GenesisConfig {
      fn build(&self) {
          // custom values are added here at the genesis block
          <AuctionIndex<T>>::put(&self.auction_index);
      }
  }
```

### Setting initial values for storage items

This step is vital to ensure that the genesis configuration works as expected in the mock runtime and as well as the main runtime.

After coupling your `runtime/lib.rs` like so:

```rust
// Configure a mock runtime to test the pallet.
frame_support::construct_runtime!(
  pub struct Runtime
    Block = Block,
    NodeBlock = Block,
    UncheckedExtrinsic = UncheckedExtrinsic,
  {
    System: frame_system,
    DoubleAuctionModule: pallet_double_auction,

    // -------------snip-----------
  }
);
```

Add an initial value for `AuctionIndex` in `node/chain_spec.rs` like so:

```rust
const AUCTION_INDEX: u64 = 10001;
```

You can then use this `AUCTION_INDEX` in `node/chain_spec.rs` like so:

```rust
GenesisConfig {
  // -------------snip-----------

  pallet_double_auction: DoubleAuctionModule {
    auction_index: AUCTION_INDEX,
  },

  // -------------snip-----------
}
```

The runtime uses the data provided in `node/chain_spec.rs` to initialize `AuctionIndex` at the genesis block.

## Going in-depth

At a glance, Substrate storage genesis configuration may appear to be an unnecessary endeavor that could be easily satisfied by declaring the initial storage item directly in the pallet. However, the moment you attempt to do this, you will realize that it is not an easy feat. This becomes especially true when you attempt to initialize a storage item with generic types.

Substrate provides a developer-friendly interface that enables us to design and implement runtime modules that can be used in different runtimes without the need to rewrite the module's configuration for different runtimes.

This interface is provided mainly by two attribute macros; `genesis_config` and `genesis_build`, and a trait `GenesisBuild` which must be used together to provide a cohesive abstraction for complex storage handling and runtime operations.

`GenesisConfig` decorated with `genesis_config` allows us to define a data type as a `struct` or `enum` for the configuration of our pallet's genesis state. `genesis_config` also enforces that `GenesisConfig` can only be used from a pallet.

`GenesisConfig` also implements low-level methods that can be leveraged by the substrate runtime to add values from `GenesisConfig` to storage (even after the genesis block).

`GenesisConfig` is defined like so:

```rust
pub struct GenesisConfig {
    pub changes_trie_config: Option<ChangesTrieConfiguration>,
    pub code: Vec<u8>,
}
```

And has an implementation block like so:

```rust
impl GenesisConfig {
	/// Direct implementation of `GenesisBuild::build_storage`.
	///
	/// Kept in order not to break dependency.
	pub fn build_storage<T: Config>(&self) -> Result<sp_runtime::Storage, String> {
		<Self as GenesisBuild<T>>::build_storage(self)
	}

	/// Direct implementation of `GenesisBuild::assimilate_storage`.
	///
	/// Kept in order not to break dependency.
	pub fn assimilate_storage<T: Config>(
		&self,
		storage: &mut sp_runtime::Storage,
	) -> Result<(), String> {
		<Self as GenesisBuild<T>>::assimilate_storage(self, storage)
	}
}
```

The methods in the `impl` block are no longer the reference standard for implementing genesis configuration using substrate FRAME and as such require the `GenesisBuild` trait. 

The `GenesisBuild` trait along with `genesis_build` macro allows us to define exactly how the genesis configuration is built for each storage item provided.

`GenesisBuild` is defined like so:

```rust
pub trait GenesisBuild<T, I = ()>: Default + MaybeSerializeDeserialize {
	/// The build function is called within an externalities allowing storage APIs.
	/// Thus one can write to storage using regular pallet storages.
	fn build(&self);

	/// Build the storage using `build` inside default storage.
	fn build_storage(&self) -> Result<sp_runtime::Storage, String> {
		let mut storage = Default::default();
		self.assimilate_storage(&mut storage)?;
		Ok(storage)
	}

	/// Assimilate the storage for this module into pre-existing overlays.
	fn assimilate_storage(&self, storage: &mut sp_runtime::Storage) -> Result<(), String> {
		sp_state_machine::BasicExternalities::execute_with_storage(storage, || {
			self.build();
			Ok(())
		})
	}
}
```

Both `build_storage` and `assimilate_storage` work exactly as in `GenesisConfig` impl.

`build` is the method of interest. Substrate uses the `build` method to directly expose storage API to a pallet at the genesis block.

In our case, substrate stores the initial storage value of `AuctionIndex` under the key provided by the storage instance. This allows a pallet with multiple instances to also have multiple variants of `AuctionIndex`.

## Summary

We used the `AuctionIndex` of a double auction pallet to demonstrate how to implement a genesis configuration using Substrate FRAME pallets and APIs. We explored the steps involved in implementing a genesis config, which allows our pallet to be used on any substrate runtime with arbitrary initial value.

We developed an understanding of:
- why genesis configurations are important and useful.
- substrate abstraction for genesis configuration.
- how to implement genesis configuration on a pallet.
- how to add initial genesis configuration values in node chain-spec.

To learn more about substrate genesis config, check out these resources:
- [Configure Genesis State](https://docs.substrate.io/reference/how-to-guides/basics/configure-genesis-state/)
- [Genesis Configuration](https://docs.substrate.io/build/genesis-configuration/)
- [GenesisConfig doc](https://crates.parity.io/frame_system/pallet/struct.GenesisConfig.html)
- [GenesisBuild doc](https://crates.parity.io/frame_support/pallet_prelude/trait.GenesisBuild.html)
