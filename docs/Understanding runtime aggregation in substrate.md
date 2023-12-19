---
tags:
  - substrate
keywords: [polkadot, substrate, associated types, generics, type parameters runtime]
description: Understanding runtime aggregation in substrate
updated: 2023-12-19
author: abdulbee
duration: 3h
level: intermediate
---

## Introduction
The guide provides a comprehensive exploration of how the Substrate runtime effectively aggregates modular logic. It delves into the essential components of substrate modules, including calls, events, and errors, which are represented as enums within each pallet. 

By examining the challenges posed by the presence of multiple enums and the potential compatibility issues they may cause, the article highlights the significance of aggregating these enums into a unified type for the runtime.



> Help us measure our progress and improve Substrate in Bits content by filling
out our living [feedback form](https://airtable.com/shr7CrrZ5zqlhWEUD).
Thank you!

## How the Substrate runtime aggregates modular logic

Every substrate module is expected to have calls, events, and errors. Some pallets may have custom origins. These are represented as enums directly on these pallets. But in some cases (like in the case of pallet calls), these enums are generated using macros. Every error, call, event, and origin are all included as variants of their respective enums for each pallet.

Because each pallet has its unique enum for these types, compatibility issues (in terms of integration and interaction) could arise if there's no system in place to aggregate all the types into one unified type for the runtime. 

To avoid these compatibility issues, each enum type for each pallet is aggregated into a single enum for the runtime, with each pallet's enum type being a variant of the runtime enum. The runtime enums are then used to generate the runtime's events, errors, and calls for all available pallets that are encoded. These enums can then be used as the associated types when implementing a pallet for the runtim.

![](https://github.com/Chondria/SiB/blob/main/Images/0_outer_enums.jpg)


Essentially, you can think of the enums of individual pallets as sub-enums, which are encompassed by the parent enums (the runtime enums). This can also be translated to mean that the pallets' enums are the inner enums, while the runtime's enums are the outer enums.

The runtime's enums are defined in frame_system

```rust
/// System configuration trait. Implemented by runtime.
	#[pallet::config(with_default)]
	#[pallet::disable_frame_system_supertrait_check]
	pub trait Config: 'static + Eq + Clone {
    /// The aggregated event type of the runtime.
		#[pallet::no_default_bounds]
		type RuntimeEvent: Parameter
			+ Member
			+ From<Event<Self>>
			+ Debug
			+ IsType<<Self as frame_system::Config>::RuntimeEvent>;

    /// The `RuntimeOrigin` type used by dispatchable calls.
		#[pallet::no_default_bounds]
		type RuntimeOrigin: Into<Result<RawOrigin<Self::AccountId>, Self::RuntimeOrigin>>
			+ From<RawOrigin<Self::AccountId>>
			+ Clone
			+ OriginTrait<Call = Self::RuntimeCall, AccountId = Self::AccountId>;

    /// The aggregated `RuntimeCall` type.
		#[pallet::no_default_bounds]
		type RuntimeCall: Parameter
			+ Dispatchable<RuntimeOrigin = Self::RuntimeOrigin>
			+ Debug
			+ From<Call<Self>>;

    // ...
    }
```

![inner_enum](https://github.com/Chondria/SiB/blob/main/Images/1_inner_enum.jpg)

Let's now take a look at how each of these types are aggregated for the runtime

## calls aggregation
All extrinsic calls in substrate modules are annotated with the  `#[pallet::call]` attribute. Every call with this attribute is included as a variant in the `Call` enum for the module, which is then included in the runtime.

```rust
#[pallet::call]
impl<T: Config> Pallet<T> {
    #[pallet::call_index(0)]
    #[pallet::weight(10_000)]
    pub fn first_extrinsic(origin: OriginFor<T>, data: u32) -> DispatchResultWithPostInfo {
        // Extrinsic call implementation
    }
    
    #[pallet::call_index(1)]
    #[pallet::weight(5_000)]
    pub fn second_extrinsic(origin: OriginFor<T>, data: String) -> DispatchResultWithPostInfo {
    // Extrinsic call implementation
    }
}
```

In the code above, the `#[pallet::call]` macro generates the `Call` enum and includes the two extrinsics above as variants.

```rust
pub enum Call {
    first_extrinsic {origin: OriginFor<T>, data: u32},
    econd_extrinsic {origin: OriginFor<T>, data: u32},
}

```

The `Into`  and `TryFrom` traits must be implemented for each module's `Call` enum, allowing it to be convertible into the overarching `RuntimeCall` type. These implementations are done automatically by the 


## Events aggregation

Every pallet has an enum that represents all the possible events that the pallet can emit.

```rust
#[pallet::event]
pub enum Event<T: Config<I>, I: 'static = ()> {
    Event_1 {},
    Event_2 {},
    Event_3 {},
}
```

The enum for each pallets are included as variants in the `RuntimeEvent` enum. The `Into`  and `TryFrom` trait must be implemented for each module's `Event` enum, allowing it to be convertible from the overarching `RuntimeEvent` type to the pallet's native `Event` type and vice-versa.


## Errors aggregation

Just like in the case of `Events`, Every pallet has an enum that represents all the possible errors that the pallet can emit.

```rust
#[pallet::error]
pub enum Error<T> {
    Error_1,
    Error_2,
    Error_3,
}
```

One or more variants of the `Error` enum can be returned when an error occurs in an extrinsic call.


## The construct_runtime! macro

The construct_runtime! macro plays a crucial role in ensuring that the types above are aggregated correctly for the runtime, by generating the code necessary to assemble these components, essentially handling the complexity of integrating the various components.

You can learn more about the `construct_runtime!` macro [here](https://polkadot.study/tutorials/substrate-in-bits/docs/Let%E2%80%99s%20distill%20the%20construct_runtime%20macro)


## conclusion

This article delves into the process of runtime aggregation in Substrate, enabling Substrate's runtime to efficiently handle diverse modules.. It outlines how Substrate modules encapsulate calls, events, and errors, each represented by unique enums. 

To avoid compatibility issues, these enums are aggregated into a single runtime enum, providing a unified type for the runtime. The `construct_runtime!` macro, which generates the necessary code to assemble these components and ensures they work together seamlessly, plays a critical role in seamlessly assembling these components, ensuring effective integration.

> Help us measure our progress and improve Substrate in Bits content by filling
out our living [feedback form](https://airtable.com/shr7CrrZ5zqlhWEUD).
Thank you!
