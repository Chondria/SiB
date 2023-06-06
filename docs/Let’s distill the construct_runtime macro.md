---
tags:
  - substrate
keywords: [polkadot, substrate, error handling, macros, pallets, rust]
description: Let’s distill the construct_runtime! macro
updated: 2023-06-06
author: abdbee
duration: 3h
level: intermediate
---

# Let’s distill the construct_runtime! macro

The **`construct_runtime!`** macro is an essential part of building a Substrate runtime. This macro is responsible for tying together the various pallets that compose a runtime, making them interact with each other, and exposing their APIs to the outside world (like RPC, extrinsics, etc.). It's like the glue that binds your blockchain together.

In this article, we’ll take a look at what happens when this macro is not provided with sufficient information required for your runtime compile, as well as in-depth information about this macro and how to configure it to make sure your runtime works as intended.

> To help us measure our progress and improve Substrate in Bits content, please fill out our living [feedback form](https://airtable.com/appc45lFGS94WumrY/tblnuIR8lSd4TX7IR/viwqMQuAR6zSDn765?blocks=hide). It will only take 2 minutes of your time. Thank you!
    
    

## Reproducing errors

### Environment and project set up

To follow along with this tutorial, ensure that you have the Rust toolchain installed.

- Visit the [substrate official documentation](https://docs.substrate.io/install/) page for the installation processes.
- Clone the project [repository](https://github.com/abdbee/construct-runtime).

```
git clone https://github.com/abdbee/construct-runtime.git

```

- Navigate into the project’s directory.

```
cd construct-runtime

```

- Run the command below to compile the node.

```
cargo build --release

```

While attempting to compile the node above, you’ll encounter an error similar to the one below:

```rust
error[E0277]: the trait bound `RuntimeEvent: From<pallet_identity::Event<Runtime>>` is not satisfied
     --> /workspace/NextSimpleStarter/construct_runtime/runtime/src/lib.rs:266:22
      |
  266 |     type RuntimeEvent = RuntimeEvent;
      |                         ^^^^^^^^^^^^ the trait `From<pallet_identity::Event<Runtime>>` is not implemented for `RuntimeEvent`
      |
      = help: the following other types implement trait `From<T>`:
                <RuntimeEvent as From<frame_system::Event<Runtime>>>
                <RuntimeEvent as From<pallet_balances::Event<Runtime>>>
                <RuntimeEvent as From<pallet_grandpa::Event>>
                <RuntimeEvent as From<pallet_sudo::Event<Runtime>>>
                <RuntimeEvent as From<pallet_template::Event<Runtime>>>
                <RuntimeEvent as From<pallet_transaction_payment::Event<Runtime>>>
  note: required by a bound in `pallet_identity::Config::RuntimeEvent`
     --> /workspace/.cargo/git/checkouts/substrate-7e08433d4c370a21/98f2e34/frame/identity/src/lib.rs:108:22
      |
  108 |         type RuntimeEvent: From<Event<Self>> + IsType<<Self as frame_system::Config>::RuntimeEvent>;
      |                            ^^^^^^^^^^^^^^^^^ required by this bound in `Config::RuntimeEvent`

  For more information about this error, try `rustc --explain E0277`.
  error: could not compile `node-template-runtime` (lib) due to previous error
```

This error is saying that the `From<pallet_identity::Event<Runtime>>` is not  implemented for `RuntimeEvent`. It's telling you that the `RuntimeEvent` type is expected to implement the `From<pallet_identity::Event<Runtime>>` trait, but it doesn't.

The `construct_runtime!` macro in Substrate creates an overarching enum type, `RuntimeEvent`, which represents all possible events that can be emitted by pallets included in your runtime. When you include a pallet, for example, `pallet_identity`, in the `construct_runtime!` macro, the macro automatically adds a variant to this `RuntimeEvent` enum for the pallet's events. The `RuntimeEvent` enum then implements the `From` trait of the specific pallet's event type (in this case, `From<pallet_identity::Event<Runtime>>`)  This trait implementation enables automatic conversion (or "upcasting") from a pallet-specific event type (`pallet_identity::Event<Runtime>`) to the generic `RuntimeEvent` type. This allows any pallet event to be treated as a generic event in the runtime.

But since `pallet_identity` is not included in the `construct_runtime!` macro, the macro does not include the `pallet_identity::Event<Runtime>` variant in the `Event` enum. Consequently, the `From<pallet_identity::Event<Runtime>>` trait is not implemented for the `RuntimeEvent` enum.

However, the implementation of `pallet_identity::Config` for your runtime expects that the `RuntimeEvent` type implements the `From<pallet_identity::Event<Runtime>>` trait. Since this is not the case, the compiler throws an error.

## Solving the error

- Go to the [lib.rs](https://github.com/abdbee/construct-runtime/blob/main/runtime/src/lib.rs) file of your node’s runtime.

- Add `pallet_identity` to the `construct_runtime!` macro.

```rust
construct_runtime!(
    pub struct Runtime where
        Block = Block,
        NodeBlock = opaque::Block,
        UncheckedExtrinsic = UncheckedExtrinsic,
    {
        // ----- *snip* ------
        Identity: pallet_identity, //<------ add this line
    }
);

```

Including `pallet_identity` in the construct_runtime macro does a couple of things.

- It adds it as a variant in the `RuntimeEvents` enum
- It implements `From<pallet_identity::Event<Runtime>>` for the `RuntimeEvents` type, which satisfies the trait requirements for `pallet_identity::Config::RuntimeEvents`

- Re-compile the node

```rust
cargo build --release
```

## Going in-depth

The **`construct_runtime!`** macro in Substrate is a declarative macro that enables you to compose your blockchain runtime by specifying a selection of individual pallets. The macro generates boilerplate code to glue these pallets together into a cohesive whole.

Each pallet declaration includes:

- The unique **`Identifier`** that represents the pallet.
- The **`path::to::pallet`**, which specifies where the pallet's definition can be found.
- An optional **`::<InstanceN>`** part if the pallet supports multiple instances.
- A comma-separated list of parts enclosed in **`{}`**. These parts, like **`Pallet`**, **`Call`**, **`Storage`**, **`Event<T>`**, **`Origin<T>`**, **`Config<T>`**, etc., represent different components of the pallet. They are used to export the pallet's functionalities correctly in the runtime and its metadata.

```rust
YourPallet: pallet_your_pallet::{Pallet, Call, Storage, Event<T>, Config<T>},
```

In this example, `YourPallet` is the unique identifier for the pallet, and `pallet_your_pallet` is the path to the pallet's definition. The parts within `{}` represent different components of the pallet that you want to include in the runtime, such as the pallet's main logic (`Pallet`), extrinsic calls (`Call`), storage (`Storage`), events (`Event<T>`), and configuration (`Config<T>`).

You can also assign an index to each pallet using `= $n`, which is used to encode the pallet's variants in types like `OriginCaller`, `Call`, and `Event`. This is particularly useful for larger-scale projects, where multiple pallets may be interacting, as it helps trace the flow of events and calls. It also allows for better error tracing and debugging, as it's immediately clear from the encoded data which pallet an event or call is associated with.

The macro is also responsible for associating your runtime’s concrete type implementations with the pallet’s types. For example, if a pallet requires a `Currency` trait with an associated type `Balance`, and you've implemented `Currency` for your runtime with `Balance` as `u32`, the `construct_runtime!` macro ensures that `Currency::Balance` in that pallet is replaced with `u32`.

For each included pallet, the `construct_runtime!` macro generates a type alias mapping the pallet to the runtime. This allows the pallet's functionalities to be accessed under the given identifier in the runtime and makes it easier to differentiate between multiple instances of the same pallet. For example, an alias for one instance of the `collective` pallet might be “Council” and another alias could be “Technical Committee”. These aliases ensure that you can easily identify and interact with the intended pallet.

The `construct_runtime!` macro also handles events emitted by the pallets. It creates a `RuntimeEvent` enum type that includes variants for each pallet's events. `RuntimeEvent` implements the `From` trait of the respective pallet's event types, allowing the events to be converted into the overarching `RuntimeEvent` type.

This enables a unified way of handling events from different pallets. However, if you remove a pallet from the `construct_runtime!` macro, the associated `From` trait implementation is also removed. Consequently, the `construct_runtime!` macro will no longer generate the variant corresponding to that pallet’s event type. This would mean that events from that pallet are no longer acknowledged by the runtime's `RuntimeEvent` enum. More crucially, the macro will not generate the `From` trait implementation for the pallet’s event type on the `RuntimeEvent` enum. This implementation is crucial as it allows events coming from the pallet to be transformed into the generic `RuntimeEvent` type.

As a result, if there is still code in your runtime that deals with such a pallet and its events directly, it will fail to compile because `RuntimeEvents` no longer acknowledges the event type of that pallet.

## Summary

In this Substrate in Bits content, we delved into the `construct_runtime!` macro, a crucial component in building a Substrate runtime. This macro facilitates the assembly of various pallets that constitute a runtime, orchestrates their interaction, and exposes their APIs for external use, thereby serving as the “adhesive” that binds a runtime together.

The article extensively explained what transpires when this macro lacks sufficient information for runtime compilation. It also provides an in-depth view of the macro's configuration and its role in ensuring smooth operation of your runtime with a step-by-step tutorial to reproduce and resolve a common error encountered when a pallet's event variant is missing in the `construct_runtime!` macro, using the `pallet_identity` as an example.

The `construct_runtime!` macro's function is broken down in detail, from pallet declaration, which includes elements like the unique Identifier, path to the pallet, optional instances, and different components, to assigning an index to each pallet for ease of debugging and tracing events. We also discussed how the macro associates the runtime's concrete type implementations with the pallet's types and handles events emitted by the pallets by generating a **`RuntimeEvent`** enum type. If a pallet is removed from the `construct_runtime!` macro, the associated `From` trait implementation is also omitted, leading to potential compilation errors.

Thus, understanding the `construct_runtime!` macro is vital for developers working with the Substrate framework, as it brings the various moving parts of the blockchain together.

To learn more about the concepts discussed in this guide, here are some resources that we recommend:

- [FRAME macros](https://docs.substrate.io/reference/frame-macros/)
- [Using macros in a custom pallet](https://docs.substrate.io/tutorials/build-application-logic/use-macros-in-a-custom-pallet/)
- [Runtime construction macros](https://docs.substrate.io/reference/frame-macros/#runtime-construction-macros)

> We’re inviting you to fill out our living [feedback form](https://airtable.com/shr7CrrZ5zqlhWEUD) to help us measure our progress and improve Substrate in Bits content. It will only take 2 minutes of your time. Thank you!
