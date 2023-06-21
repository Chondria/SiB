---
tags:
  - substrate
keywords: [polkadot, substrate, traits, config, pallets, rust]
description: Substrate Pallets Configuration, An Error-based Approach
updated: 2023-06-21
author: abdbee
duration: 3h
level: intermediate
---

# Substrate Pallets Configuration: An Error-based Approach

When implementing a pallet for your runtime, there are things to be kept in place for your runtime to work as intended. One of those things is ensuring that a concrete type is defined for all the associated types in the Pallet’s Config trait and that all the concrete types used for the runtime configuration are bound by traits specified by the pallet. 

In this guide, we’ll take an error-based approach to understand what happens when you don’t configure the pallets correctly for your runtime.

> To help us measure our progress and improve Substrate in Bits content, please fill out our living [feedback form](https://airtable.com/appc45lFGS94WumrY/tblnuIR8lSd4TX7IR/viwqMQuAR6zSDn765?blocks=hide). It will only take 2 minutes of your time. Thank you!


## Reproducing errors

### Environment and project setup

To follow along with this tutorial, ensure that you have the Rust toolchain installed.

- Visit the [substrate official documentation](https://docs.substrate.io/install/) page for the installation processes.
- Clone the project [repository](https://github.com/abdbee/sibchain/tree/pallet_implementation).

```
git clone -b pallet_implementation --single-branch https://github.com/abdbee/sibchain.git
```

- navigate to the project’s directory

```
cd sibchain
```

- Run the command below to compile the node

```
cargo build --release
```

While attempting to compile the node above, you’ll encounter an error similar to the one below:

```rust
error[E0046]: not all trait items implemented, missing: `ExistentialDeposit`
     --> /workspace/NextSimpleStarter/sibchain/runtime/src/lib.rs:233:1
      |
  233 | impl pallet_balances::Config for Runtime {
      | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ missing `ExistentialDeposit` in implementation
      |
      = help: implement the missing item: `type ExistentialDeposit = Type;`

  For more information about this error, try `rustc --explain E0046`.
  error: could not compile `node-template-runtime` due to previous error
```

The error above is saying that a concrete type for `ExistentialDeposit` is missing in the runtime implementation of the `Balances` pallet. 

Since this type was included in the `Balances` pallet’s `Config` trait, a concrete type has to be assigned to it in the runtime’s implementation for the node to compile. The type used has to be bound by the traits specified in the pallet’s `Config` trait, which in this case is `GetSelf::Balance`

```rust
#[pallet::config]
pub trait Config<I: 'static = ()>: frame_system::Config {
    // <----- Other types <---------
    #[pallet::constant]
    type ExistentialDeposit : Get<Self::Balance>;
    // <----- Other types <---------
}
```

In this case, bounding `GetSelf::Balance` to the `ExistentialDeposit` associated type makes `ExistentialDeposit` a configuration parameter that is expected to return a `Balance` type. This `Balance` type must in turn satisfy all the trait bounds mentioned in the definition of `type Balance` in the pallet’s `Config` trait.

## Solving the error

- Navigate to the node’s runtime.

```
cd runtime/src/lib.rs
```

- Go to where the `Balance` pallet’s `Config` trait was implemented for the runtime, and define a concrete type for the `ExistentialDeposit` associated type.

```rust
/// Existential deposit. 
pub const EXISTENTIAL_DEPOSIT: u128 = 500; //<--- Add this line

impl pallet_balances::Config for Runtime {
    // Other associated type assignments
    type ExistentialDeposit = ConstU128<EXISTENTIAL_DEPOSIT>; //<--- Add this line
    // Other associated type assignments
}
```

In the code above, `ExistentialDeposit` is defined as a constant value of type `u128`. When `ExistentialDeposit` is accessed via its associated getter function, the value of `EXISTENTIAL_DEPOSIT` is returned. 

## Going in-depth

In Substrate, each pallet defines a `Config` trait. This trait is a way to define configuration settings and types specific to the pallet, which can then be customized when the pallet is integrated into a runtime. These traits often have associated types that let you define placeholders for types that will be specified when the trait is implemented.

In the snippet below:

```rust
#[pallet::config]
pub trait Config<I: 'static = ()>: frame_system::Config {
    /// The overarching event type.
    type RuntimeEvent: From<Event<Self, I>>
        + IsType<<Self as frame_system::Config>::RuntimeEvent>;

    /// Weight information for extrinsics in this pallet.
    type WeightInfo: WeightInfo;

    /// The balance of an account.
    type Balance: Parameter
        + Member
        + AtLeast32BitUnsigned
        + Codec
        + Default
        + Copy
        + MaybeSerializeDeserialize
        + Debug
        + MaxEncodedLen
        + TypeInfo
        + FixedPointOperand;
    //<---- snip ------->
}
```

`RuntimeEvent`, `WeightInfo`, and `Balance` are associated types of the `Config` trait. They are declared, but not defined, within the trait. The concrete types for these will need to be provided by any runtime which uses this pallet.

When integrating the pallet into your runtime, your code will look like this:

```rust
impl pallet_balances::Config for Runtime {
	type RuntimeEvent = RuntimeEvent;
	type Balance = u128;
	type WeightInfo = //weight definition
}
```

This is where the actual concrete types are provided for the associated types declared in the pallet's `Config` trait.

**Here’s what is happening in this case:**

- **`type RuntimeEvent = RuntimeEvent;`** This is setting the `RuntimeEvent` type for the **`pallet_balances`** to be the **`RuntimeEvent`** enum defined in the runtime. This allows the pallet to deposit events that can be recognized and handled by the runtime's event system.
- **`type Balance = u128;`** This sets the **`Balance`** type to **`u128`**. This is how the pallet will represent account balances.

Note, however, that such implementations can also include constants and more complex configurations.

This pattern is a key design philosophy in Substrate, as it allows each runtime to specify its own types and configuration settings for each pallet, thereby enabling pallets to be reusable across different blockchains with different needs.

Remember, when defining the associated type, you have to make sure it fulfills all trait bounds that have been defined in the pallet's `Config` trait. For example, if `Balance` in a pallet's `Config` trait is defined as a `Parameter + Member + AtLeast32BitUnsigned + Default + Copy`, the concrete type that you provide must satisfy all of these traits. If not, you will get a compile-time error.

 If you’re not sure of which concrete type to use or don’t have a custom type that has the necessary traits implemented for it, you can check and use the substrate [node implementations](https://github.com/paritytech/substrate/tree/master/bin/node) of these pallets’ `Config` traits to save you time.

In cases where a particular type will not be used or where the default behavior is intended, the “unit type” (`()`) can be used. This can also be used as a placeholder to allow your code to run before you assign a concrete type. Note however that in production, you should take the time to define these types properly to match your chain's requirements.

## Summary

In this Substrate in Bits content, we explore what could go wrong if pallets are not implemented wholly for your runtime. Crucially, it's necessary to ensure that all required types for a pallet are implemented, and all concrete types used for runtime configuration are bound by traits specified by the pallet. Incorrect configuration can lead to errors, which we illustrate using an example in this guide.

Diving deeper, we explain that in Substrate, each pallet defines a Config trait. These traits often include associated types bound by specific traits. The concrete types for these associated types need to be specified when the trait is implemented.

This modular approach is a key design philosophy in Substrate, allowing each runtime to specify its own types and configuration settings for each pallet, thereby making the pallets reusable across different blockchains with diverse needs.

Lastly, it's worth noting that in cases where a particular type will not be used, or default behavior is intended, the “unit type” (`()`) can be used. This can be useful as a placeholder to allow your code to run before you design and assign a concrete type. However, in a production setting, you should take the time to define these types properly to match your chain's requirements.

To learn more about the concepts discussed in this guide, here are some resources that we recommend:

- [Configure runtime constants](https://docs.substrate.io/reference/how-to-guides/basics/configure-runtime-constants/)
- [Add a pallet to the runtime.](https://docs.substrate.io/tutorials/build-application-logic/add-a-pallet/)

> To help us measure our progress and improve Substrate in Bits content, please fill out our living [feedback form](https://airtable.com/appc45lFGS94WumrY/tblnuIR8lSd4TX7IR/viwqMQuAR6zSDn765?blocks=hide). It will only take 2 minutes of your time. Thank you!
