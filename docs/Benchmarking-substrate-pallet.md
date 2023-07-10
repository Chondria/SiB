---
tags:
  - substrate
keywords: [polkadot, substrate, pallets, benchmarking, weights, rust]
description: Benchmarking substrate pallets.
updated: 2023-07-10
author: cenwadike
duration: 4h
level: intermediate
---

# Benchmarking substrate pallet
Because blockchain runtime engineers are empowered by Substrate to build custom logic that runs as part of the runtime, they are obligated to ensure that their blockchain behaves correctly across nodes with a minimum set of hardware requirements.

Substrate enables runtime engineers to provide deterministic and verifiable guarantees that a minimum hardware requirement is sufficient to participate in the blockchain network as a node by providing sophisticated and comprehensive tools for benchmarking a runtime and its pallets.

In this guide, we will break down important steps involved in substrate pallet benchmarking using a problem-based approach. We will also highlight some of the internal processes of benchmarking a substrate pallet and substrate benchmarking components that can be leveraged to benchmark a substrate pallet.

>Help us measure our progress and improve Substrate in Bits content by filling out our living [feedback form](https://airtable.com/appc45lFGS94WumrY/tblnuIR8lSd4TX7IR/viwqMQuAR6zSDn765?blocks=hide). Thank you!

## Reproducing errors

### Environment and project setup

To follow along with this tutorial, ensure that you have the Rust toolchain installed.

- Visit the [substrate official documentation](https://docs.substrate.io/install/) page for the installation processes.

- Clone the project [repository](https://github.com/cenwadike/book-archiver-node).

```rust
git clone https://github.com/cenwadike/book-archiver-node
```

- Navigate into the project’s directory.

```
cd book-archiver-node
```

- Checkout to faulty commit

```
git checkout 9437d91fc9ae654
```

- Run the command below to build the node.

```
cargo build --release
```

- Run the command below to run the pallet benchmark test.

```
cargo test --package pallet-template --features runtime-benchmarks
```

- Run the command below to run runtime benchmarking.

```
cargo build --package node-template --release --features runtime-benchmarks
```

- Run the command below to generate pallet weights.

```
./target/release/node-template benchmark pallet \
--chain dev \
--pallet pallet_template \
--extrinsic '*' \
--steps 50 \
--repeat 20 \
--output pallets/template/src/weights.rs
```

While attempting to run the last command above, you will....encounter no error.

To trigger the error, let us add generated benchmark weights to the pallet.

- open `pallets/template/src/lib.rs` in your code editor

- import generated weights at the top of `pallets/template/src/lib.rs` like so:

```rust
//---------snip-----------//
#[cfg(test)]
mod tests;

#[cfg(feature = "runtime-benchmarks")]
mod benchmarking;
pub mod weights;    // <---- add this 
pub use weights::*; // <---- add this 

#[frame_support::pallet]
pub mod pallet {
    //---------snip-----------//
}

```

- add `WeightInfo` into pallet `Config` trait like so:

```rust
//---------snip-----------//
#[cfg(test)]
mod tests;

#[cfg(feature = "runtime-benchmarks")]
mod benchmarking;
pub mod weights;    // <---- add this line
pub use weights::*; // <---- add this line

#[frame_support::pallet]
pub mod pallet {
    //---------snip-----------//

    #[pallet::config]
	pub trait Config: frame_system::Config {
		/// Because this pallet emits events, it depends on the runtime's definition of an event.
		type RuntimeEvent: From<Event<Self>> + IsType<<Self as frame_system::Config>::RuntimeEvent>;
        // Pallet extrinsic weight info from benchmark 
		type WeightInfo: WeightInfo;   //<------------------ add this line
	}

    //---------snip-----------//

}

```

- assign generated weight to pallet extrinsic like so:

```rust
//---------snip-----------//
#[cfg(test)]
mod tests;

#[cfg(feature = "runtime-benchmarks")]
mod benchmarking;
pub mod weights;    // <---- add this line
pub use weights::*; // <---- add this line

#[frame_support::pallet]
pub mod pallet {
    //---------snip-----------//

    #[pallet::config]
	pub trait Config: frame_system::Config {
		/// Because this pallet emits events, it depends on the runtime's definition of an event.
		type RuntimeEvent: From<Event<Self>> + IsType<<Self as frame_system::Config>::RuntimeEvent>;
        // Pallet extrinsic weight info from benchmark 
		type WeightInfo: WeightInfo;   // <------------------ add this line
	}

    //---------snip-----------//

    #[pallet::call]
	impl<T: Config> Pallet<T> {
		#[pallet::call_index(1)]
		#[pallet::weight(T::WeightInfo::archive_book())]  // <----- modify this line
		pub fn archive_book(
			origin: OriginFor<T>,
			title: Vec<u8>,
			author: Vec<u8>,
			url: Vec<u8>,
		) -> DispatchResult {
			//---------snip-----------//

			// Store book summary in archive
			ArchiveStore::<T>::insert(&book_hash, book_summary);

			// Emit an event that the book was archived.
			Self::deposit_event(Event::BookArchived { who: signer });

			Ok(())
		}
	}

}

```

- Run the command below to build the node.

```
cargo build --release
```

If you followed the above steps correctly you will encounter an error like the one below:

```bash
error[E0433]: failed to resolve: use of undeclared crate or module `pallet_template`
  --> pallets/template/src/weights.rs:37:31
   |
37 | impl<T: frame_system::Config> pallet_template::WeightInfo for WeightInfo<T> {
   |                               ^^^^^^^^^^^^^^^ use of undeclared crate or module `pallet_template`

error[E0404]: expected trait, found struct `WeightInfo`
  --> pallets/template/src/lib.rs:54:17
   |
54 |         type WeightInfo: WeightInfo;
   |                       ^^^^^^^^^^ not a trait
   |
help: consider importing this trait instead
   |
38 |     use frame_system::WeightInfo;
```

## Solving the error

The compiler throws two error messages at us that cumulatively draw us to a root cause. *`pallet_template`* was undeclared in `pallets/template/src/weights.rs` and the pallet `Config` `WeightInfo` type was assigned a `struct` in place of a `trait`. 

Both errors are rooted in autogenerated *`pallets/template/src/weights.rs`* file

A careful inspection of `pallets/template/src/weights.rs` shows that it contains a struct `WeightInfo` that implements weight generating function for our pallet extrinsic like so:

```rust
    //---------snip-----------//

pub struct WeightInfo<T>(PhantomData<T>);

impl<T: frame_system::Config> pallet_template::WeightInfo for WeightInfo<T> {
	fn archive_book() -> Weight {
        //---------snip-----------//
	}
}
```

This setup is not compatible with the substrate pallet `Config`. And logically a fix would be to supply a trait that implements weight generating function for our pallet.

We could re-write the weights file like so: 
```rust
    //---------snip-----------//

pub trait WeightInfo {
	fn archive_book() -> Weight;
}

pub struct SubstrateWeight<T>(PhantomData<T>);

impl<T: frame_system::Config> WeightInfo for SubstrateWeight<T> {
    fn archive_book() -> Weight {
        Weight::from_parts(5_000_000,
        3471)
        .saturating_add(T::DbWeight::get().reads(1_u64))
        .saturating_add(T::DbWeight::get().writes(1_u64))
    }
}
```
CAUTION: DO NOT MANUALLY UPDATE OR WRITE IN A *`pallets/pallet-your-pallet/wieghts.rs`* UNLESS YOU REALLY KNOW WHAT YOU ARE DOING

The more appropriate way to update *`pallets/template/src/weights.rs`* to suit our needs will be to autogenerate weights using a similar format above.

To provide a template for the format, we have to add *`scripts/frame-weight-template.hbs`* file to our node root directory like so:

```bash
touch scripts/frame-weight-template.hbs
```

- Open *`scripts/frame-weight-template.hbs`* in your code editor 
- and copy the [frame-weight-template](https://github.com/paritytech/substrate/blob/master/.maintain/frame-weight-template.hbs) into *`scripts/frame-weight-template.hbs`*

We can now regenerate our pallet weights with the correct format following these steps:

1. First, we comment out the code that triggered this error so that we can successfully compile our runtime and generate weights for our extrinsic

- open `pallets/template/src/lib.rs` in your code editor

- remove generated weights at the top of `pallets/template/src/lib.rs` like so:

```rust
//---------snip-----------//
#[cfg(test)]
mod tests;

#[cfg(feature = "runtime-benchmarks")]
mod benchmarking;
// pub mod weights;    // <---- comment this line
// pub use weights::*; // <---- comment this line

#[frame_support::pallet]
pub mod pallet {
    //---------snip-----------//
}

```

- remove `WeightInfo` from pallet `Config` trait like so:

```rust
//---------snip-----------//
#[cfg(test)]
mod tests;

#[cfg(feature = "runtime-benchmarks")]
mod benchmarking;
// pub mod weights;    // <---- comment this line
// pub use weights::*; // <---- comment this line

#[frame_support::pallet]
pub mod pallet {
    //---------snip-----------//

    #[pallet::config]
	pub trait Config: frame_system::Config {
		/// Because this pallet emits events, it depends on the runtime's definition of an event.
		type RuntimeEvent: From<Event<Self>> + IsType<<Self as frame_system::Config>::RuntimeEvent>;
        // Pallet extrinsic weight info from benchmark 
		// type WeightInfo: WeightInfo;   //<------------------ comment this line
	}

    //---------snip-----------//

}

```

- use placeholder weight on pallet extrinsic like so:

```rust
//---------snip-----------//
#[cfg(test)]
mod tests;

#[cfg(feature = "runtime-benchmarks")]
mod benchmarking;
// pub mod weights;    // <---- comment this line
// pub use weights::*; // <---- comment this line

#[frame_support::pallet]
pub mod pallet {
    //---------snip-----------//

    #[pallet::config]
	pub trait Config: frame_system::Config {
		/// Because this pallet emits events, it depends on the runtime's definition of an event.
		type RuntimeEvent: From<Event<Self>> + IsType<<Self as frame_system::Config>::RuntimeEvent>;
        // Pallet extrinsic weight info from benchmark 
		// type WeightInfo: WeightInfo;   // <------------------ comment this line
	}

    //---------snip-----------//

    #[pallet::call]
	impl<T: Config> Pallet<T> {
		#[pallet::call_index(1)]
        #[pallet::weight(1_000_000)]
		// #[pallet::weight(T::WeightInfo::archive_book())]  // <----- comment this line
		pub fn archive_book(
			origin: OriginFor<T>,
			title: Vec<u8>,
			author: Vec<u8>,
			url: Vec<u8>,
		) -> DispatchResult {
			//---------snip-----------//

			// Store book summary in archive
			ArchiveStore::<T>::insert(&book_hash, book_summary);

			// Emit an event that the book was archived.
			Self::deposit_event(Event::BookArchived { who: signer });

			Ok(())
		}
	}
}

```

Confirm that everything still compiles like so:
```bash
cargo check
```

To generate weights for our pallet following the required format follow these steps:

- Build the node using the command below.

```bash
cargo build --release
```

- Run the command below to run the pallet benchmark test.

```bash
cargo test --package pallet-template --features runtime-benchmarks
```

- Run the command below to run runtime benchmarking.

```bash
cargo build --package node-template --release --features runtime-benchmarks
```

- Run the command below to generate pallet weights.

```bash
./target/release/node-template benchmark pallet \
--chain dev \
--pallet pallet_template \
--extrinsic '*' \
--steps 50 \
--repeat 20 \
--template ./scripts/frame-weight-template.hbs \
--output pallets/template/src/weights.rs
```

If you have followed the steps above correctly, you will automatically update the format of *`pallets/template/weights.rs`* 

To use the generated weights in our pallet you simply uncomment previously commented code in the preceeding steps.
You can take it as a challenge or follow the steps below for guidance:

- import generated weights at the top of `pallets/template/src/lib.rs` like so:

```rust
//---------snip-----------//
#[cfg(test)]
mod tests;

#[cfg(feature = "runtime-benchmarks")]
mod benchmarking;
pub mod weights;    // <---- add this 
pub use weights::*; // <---- add this 

#[frame_support::pallet]
pub mod pallet {
    //---------snip-----------//
}

```

- add `WeightInfo` into pallet `Config` trait like so:

```rust
//---------snip-----------//
#[cfg(test)]
mod tests;

#[cfg(feature = "runtime-benchmarks")]
mod benchmarking;
pub mod weights;    // <---- add this line
pub use weights::*; // <---- add this line

#[frame_support::pallet]
pub mod pallet {
    //---------snip-----------//

    #[pallet::config]
	pub trait Config: frame_system::Config {
		/// Because this pallet emits events, it depends on the runtime's definition of an event.
		type RuntimeEvent: From<Event<Self>> + IsType<<Self as frame_system::Config>::RuntimeEvent>;
        // Pallet extrinsic weight info from benchmark 
		type WeightInfo: WeightInfo;   //<------------------ add this line
	}

    //---------snip-----------//

}

```

- assign generated weight to pallet extrinsic like so:

```rust
//---------snip-----------//
#[cfg(test)]
mod tests;

#[cfg(feature = "runtime-benchmarks")]
mod benchmarking;
pub mod weights;    // <---- add this line
pub use weights::*; // <---- add this line

#[frame_support::pallet]
pub mod pallet {
    //---------snip-----------//

    #[pallet::config]
	pub trait Config: frame_system::Config {
		/// Because this pallet emits events, it depends on the runtime's definition of an event.
		type RuntimeEvent: From<Event<Self>> + IsType<<Self as frame_system::Config>::RuntimeEvent>;
        // Pallet extrinsic weight info from benchmark 
		type WeightInfo: WeightInfo;   // <------------------ add this line
	}

    //---------snip-----------//

    #[pallet::call]
	impl<T: Config> Pallet<T> {
		#[pallet::call_index(1)]
		#[pallet::weight(T::WeightInfo::archive_book())]  // <----- modify this line
		pub fn archive_book(
			origin: OriginFor<T>,
			title: Vec<u8>,
			author: Vec<u8>,
			url: Vec<u8>,
		) -> DispatchResult {
			//---------snip-----------//

			// Store book summary in archive
			ArchiveStore::<T>::insert(&book_hash, book_summary);

			// Emit an event that the book was archived.
			Self::deposit_event(Event::BookArchived { who: signer });

			Ok(())
		}
	}

}

```

- open *`runtime/src/lib.rs` in your code editor

- add `WeightInfo` for pallet `Config` in the runtime like so:

```rust
/// Configure the pallet_template in pallets/template.
impl pallet_template::Config for Runtime {
	type RuntimeEvent = RuntimeEvent;
	type WeightInfo = pallet_template::weights::SubstrateWeight<Runtime>; // <----- add this line
}
```

- Run the command below to build the node.

```
cargo build --release
```

- add `WeightInfo` in pallet `Config` in the mock runtime like so:
```rust
/// Configure the pallet_template in pallets/template.
impl pallet_template::Config for Test {
	type RuntimeEvent = RuntimeEvent;
	type WeightInfo = (); // <----- add this line
}
```

Both test and compilation should run successfully.


## Going in-depth

### Under the hood
Substrate pallet benchmarking process is a series of steps that are carried out using the runtime to mimic the execution of extrinsics and generate weights based on the specified hardware configuration.

A deep dive into Substrate extrinsic is extensively described by Shawn T in this substrate seminar [recording](https://www.google.com/search?q=substrate+benchmark+deep+dive&oq=substrate+ben&gs_lcrp=EgZjaHJvbWUqBggAEEUYOzIGCAAQRRg7MgYIARBFGEAyBggCEEUYOTIGCAMQIxgnMgYIBBBFGDsyBggFEEUYPDIGCAYQRRg8MgYIBxBFGDzSAQg5NTE1ajBqN6gCALACAA&sourceid=chrome&ie=UTF-8#fpstate=ive&vld=cid:d5e77f79,vid:Qa6sTyUqgek).


## Summary
In this guide, we explored a common problem when benchmarking substrate pallets. We also developed an understanding of:
- important steps involved in benchmarking substrate pallets.
- how to generate weights for a pallet's extrinsic.
- how to add benchmark-generated weights to our pallet.
- how substrate pallet benchmarking works under the hood.

Benchmarking in Substrate goes beyond pallet benchmarking. Literally every major component can be benchmarked to offer safety guarantees, this guide only discussed the surface Substrate pallet benchmarking.

To learn more about testing in substrate, check out these resources:
- [Pallet benchmarking](https://docs.substrate.io/reference/how-to-guides/weights/add-benchmarks/)
- [Substrate benchmark](https://docs.substrate.io/test/benchmark/)
- [Benchmarking deep dive](https://www.youtube.com/watch?v=Qa6sTyUqgek)
- [FRAME Benchmarking framework deep dive](https://www.youtube.com/watch?v=lgghDQpXgKk)
- [FRAME benchmarking crate](https://docs.rs/frame-benchmarking/latest/frame_benchmarking/v2/index.html)


> We’re inviting you to fill out our living [feedback form](https://airtable.com/shr7CrrZ5zqlhWEUD) to help us measure our progress and improve Substrate in Bits content. It will only take 2 minutes of your time. Thank you!



