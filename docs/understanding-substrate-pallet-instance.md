---
tags:
  - substrate
keywords: [polkadot, substrate, pallets, instance, runtime, rust]
description: Understanding substrate pallet instance.
updated: 2023-08-22
author: cenwadike
duration: 3h
level: intermediate
---

# Understanding substrate pallet instance
It is not uncommon to run multiple instances of a pallet at the same time on a runtime. This is demonstrated with the [collective pallet](https://paritytech.github.io/substrate/master/pallet_collective/index.html) that has multiple instances running on the Polkadot relay chain. Multiple instances make it possible to reuse a pallet without re-implementing it for a runtime. This allows the implementation of many use cases including having multiple tokens for a liquidity pool pallet with separate supply and distribution.

Substrate enables runtime developers a soft landing, providing valuable traits to implement an instantiable pallet. Substrate also provides smooth handling of unique storage for different instances of a pallet.

In this guide, we will learn how to make a pallet instantiable and highlight how we can test and implement an instantiable pallet on a substrate runtime.

>Help us measure our progress and improve Substrate in Bits content by filling out our living [feedback form](https://airtable.com/shr7CrrZ5zqlhWEUD). Thank you!

## Reproducing setup

### Project setup

To follow along, ensure you have the Rust toolchain installed.

- Visit the [substrate official documentation](https://docs.substrate.io/install/) page for the installation processes.

- Clone the project [repository](https://github.com/cenwadike/double-auction-node).

```rust
git clone https://github.com/cenwadike/double-auction-node
```

- Navigate into the project’s directory.

```
cd double-auction-node
```

- Run the command below to build the pallet.

```
cargo build --release
```


## Getting some context

The setup below is a substrate pallet that implements [double-auction](https://en.wikipedia.org/wiki/Double_auction) for electrical energy.

The seller gets matched (and their electricity is sold to the highest bidder) when the auction period is over.

Auctions to be executed are stored in *`AuctionsExecutionQueue`*. 
When an auction is over, it is taken from the *`AuctionsExecutionQueue`* and matched to the highest bidder.

Use the setup above is used as a reference if you get stuck.


## Making Pallet instantiable
Unlike regular pallets that have a single instance on a runtime, instantiable pallets must provide clues to the runtime. 
This clue is provided by adding an _instantiable_ trait *`I`*.

The trait *`I`* is used to provide a lifetime for a pallet’s generic types and its configuration T *T*.

We can add the generic type *`I`* to the `Pallet` struct like so:

```rust
	#[pallet::pallet]
	pub struct Pallet<T, I = ()>(PhantomData<(T, I)>);
```
This defines an empty `Pallet` struct on which we can define a pallet configuration and implement extrinsics.


### Making Pallet Config instantiable
To implement a `Config` trait compatible with our instantiable `Pallet`, we can need to add the generic type *I* like so:

```rust
  #[pallet::config]
	pub trait Config<I: 'static = ()>: frame_system::Config { // <-- notice I: `static = () 
		type RuntimeEvent: From<Event<Self, I>>
			+ IsType<<Self as frame_system::Config>::RuntimeEvent>;

		// ---------------------snip---------------------------
	}
```
This approach defines a `'static` lifetime on the generic type *I*. 


### Making Storage items instantiable
Because storage items use types defined by the `Config` trait, we also need to add the generic type *I* to the storage definition.

Using the *`AuctionsExecutionQueue`* we can use the types defined in the `Config` trait like so:

```rust
	#[pallet::storage]
	#[pallet::getter(fn auction_end_time)]
	pub(super) type AuctionsExecutionQueue<T: Config<I>, I: 'static = ()> = StorageDoubleMap<
		_,
		Twox64Concat,
		BlockNumberFor<T>,
		Blake2_128Concat,
		T::AuctionId,
		(),
		OptionQuery,
	>;
```

Notice that we applied the `'static` lifetime to the *`AuctionsExecutionQueue`* type definition.


### Events and Errors for instantiable pallet
Similar to storage items, errors, and events are unique to a `Pallet`. `Events` can be defined for an instantiable `Pallet` like so:

```rust 
  #[pallet::event]
  #[pallet::generate_deposit(pub(super) fn deposit_event)]
  pub enum Event<T: Config<I>, I: 'static = ()> {
    AuctionCreated {
            auction_id: T::AuctionId,
            seller_id: T::AccountId,
            energy_quantity: u128,
            starting_price: u128,
    },

    // ------------snip--------------
  }
```

In similar vein Errors can be defined like so:

```rust
#[pallet::error]
	pub enum Error<T, I = ()> {
		AuctionDoesNotExist,

		AuctionIsOver,

		InsuffficientAttachedDeposit,
	}
```
Notice that no lifetime is required for `Error`.


### Genesis config for instantiable pallet
Unlike the genesis configuration for a previous [guide](#/), the genesis configuration using types from the `Config` trait must the generic type *I* with a static lifetime like so:

```rust 
	#[pallet::genesis_config]
	pub struct GenesisConfig<T: Config<I>, I: 'static = ()> {
		pub auction_index: T::AuctionId,
	}

	impl<T: Config<I>, I: 'static> Default for GenesisConfig<T, I> {
		fn default() -> Self {
			Self { auction_index: Default::default() }
		}
	}

	#[pallet::genesis_build]
	impl<T: Config<I>, I: 'static> BuildGenesisConfig for GenesisConfig<T, I> {
		fn build(&self) {
			let initial_id = self.auction_index;
			<AuctionIndex<T, I>>::put(initial_id);
		}
	}
```
Notice how differently *I* is used on *GenesisConfig* struct like so ```GenesisConfig<T: Config<I>, I: 'static = ()>``` vs on the `impl` like so ```impl<T: Config<I>, I: 'static>```.

## Adding instantiable pallet to runtime
The code snippet below demonstrates how we can implement multiple instances of the [`pallet_balances`](https://github.com/paritytech/substrate/blob/master/frame/assets/src/lib.rs).

```rust
// ----------------snip----------

type MainToken = pallet_balances::Instance1;
impl pallet_balances::Config<MainToken> for Runtime {
	type MaxLocks = frame_support::traits::ConstU32<1024>;
	type MaxReserves = ();
	type ReserveIdentifier = [u8; 8];
	type Balance = u128;
	type Event = Event;
	type DustRemoval = ();
	type ExistentialDeposit = Self::ExistentialDeposit;
	type AccountStore = System;
	type WeightInfo = pallet_balances::weights::SubstrateWeight<Runtime>;
}

type DerivativeToken = pallet_balances::Instance2;
impl pallet_balances::Config<DerivativeToken> for Runtime {
	type MaxLocks = frame_support::traits::ConstU32<1024>;
	type MaxReserves = ();
	type ReserveIdentifier = [u8; 8];
	type Balance = u128;
	type Event = Event;
	type DustRemoval = ();
	type ExistentialDeposit = Self::ExistentialDeposit;
	type AccountStore = System;
	type WeightInfo = pallet_balances::weights::SubstrateWeight<Runtime>;
}
```

In comparison a single instance of [`pallet_balances`](https://github.com/paritytech/substrate/blob/master/frame/assets/src/lib.rs) like so:
```rust
impl pallet_balances::Config for Runtime {
	type MaxLocks = ConstU32<50>;
	type MaxReserves = ();
	type ReserveIdentifier = [u8; 8];
	/// The type for recording an account's balance.
	type Balance = Balance;
	/// The ubiquitous event type.
	type RuntimeEvent = RuntimeEvent;
	type DustRemoval = ();
	type ExistentialDeposit = ConstU128<EXISTENTIAL_DEPOSIT>;
	type AccountStore = System;
	type WeightInfo = pallet_balances::weights::SubstrateWeight<Runtime>;
	type FreezeIdentifier = ();
	type MaxFreezes = ();
	type RuntimeHoldReason = ();
	type MaxHolds = ();
}
```

## Common pitfalls when using instantiable pallets
Because blockchain uses unique prefixes to keep track of different storage, care must be taken to prevent two or more instances writing to the same storage item or the storage `trie`.

A way to avoid this issue is to assign a unique prefix to each instance of a `Pallet` as shown with `pallet_balances` [here](#adding-instantiable-pallet-to-runtime)


## Summary

We transformed a regular pallet into an instantiable one by adding the generic type *`I`* on different pallet components. 
We gained an understanding of how instantiable pallets are handled by the substrate runtime and used this understanding to highlight error-prone coupling for instantiable pallets.

We developed an understanding of:
- what the *`I`* generic type is and how to use it on a pallet.
- how to couple an instantiable pallet on a substrate runtime.
- how substrate runtime handles multiple instances of a pallet.
- common pitfalls when using instantiable pallets.


To learn more about instantiable pallets, check out these resources:
- [Pallet](https://paritytech.github.io/substrate/master/frame_system/pallet/index.html)
- [Collective Pallet](https://)
- [Balances Pallet](https://)
- [Polkadot Collective](https://)






