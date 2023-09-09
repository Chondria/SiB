---
tags:
  - substrate
keywords: [polkadot, substrate, error handling, constants, Get trait, pallets]
description: Dealing with runtime constants
updated: 2023-08-28
author: abdbee
duration: 3h
level: intermediate
---

# Dealing with runtime constants

Constants in Substrate runtimes serve as fixed, unchangeable values that
provide configuration data to the various components and pallets. These
constants are set at compile-time and typically use the `parameter_types!`
macro for their declaration. This allows them to benefit from type checking and
are embedded in the metadata, making them easily accessible and inspectable
from external tools or interfaces.

In this Substrate-in-Bits tutorial, we’ll take a problem-based approach to
explore why runtime constants are declared within the `parameter_types!` macro,
and why declaring an external constant can lead to issues/complications such as
exclusion from the chain's metadata, manual implementation of the `Get` trait,
and increased risk of runtime errors.

To help us measure our progress and improve Substrate in Bits content, please
fill out our living [feedback form](https://airtable.com/shr7CrrZ5zqlhWEUD). It
will only take 2 minutes of your time. Thank you!

## Reproducing errors[](https://polkadot.study/tutorials/substrate-in-bits/docs/Substrate%20Pallets%20Configuration:%20An%20Error-based%20Approach#reproducing-errors)

### Environment and project setup[](https://polkadot.study/tutorials/substrate-in-bits/docs/Substrate%20Pallets%20Configuration:%20An%20Error-based%20Approach#environment-and-project-setup)

To follow along with this tutorial, ensure that you have the Rust toolchain
installed.

- Visit the [substrate official documentation](https://docs.substrate.io/install/)
page for the installation processes.

- Clone the project [repository](https://github.com/abdbee/sibchain/tree/dealing-with-runtime-constants)

```bash
git clone -b dealing-with-runtime-constants --single-branch https://github.com/abdbee/sibchain.git
```

- navigate to the project’s directory

```bash
cd sibchain

```

- Run the command below to compile the node

```bash
cargo build --release

```

While attempting to compile the node above, you’ll encounter an error similar
to the one below:

```bash
error[E0573]: expected type, found constant `MaxAmount`
     --> /Users/abdulbee/Desktop/OpenSource/sibchain/runtime/src/lib.rs:278:19
      |
  278 |     type MaxAmount = MaxAmount;
      |                      ^^^^^^^^^ help: you might have meant to use the associated type: `Self::MaxAmount`

  error[E0152]: duplicate lang item in crate `std`: `panic_impl`.
    |
    = note: the lang item is first defined in crate `sp_io` (which `sp_application_crypto` depends on)
    = note: first definition in `sp_io` loaded from /Users/abdulbee/Desktop/OpenSource/sibchain/target/release/wbuild/node-template-runtime/target/wasm32-unknown-unknown/release/deps/libsp_io-1fb5e909d9013cdb.rmeta
    = note: second definition in `std` loaded from /Users/abdulbee/.rustup/toolchains/stable-aarch64-apple-darwin/lib/rustlib/wasm32-unknown-unknown/lib/libstd-434f4a63fb8fd380.rlib

  Some errors have detailed explanations: E0152, E0573.
  For more information about an error, try `rustc --explain E0152`.
  error: could not compile `node-template-runtime` (lib) due to 2 previous errors
```

This error is occurring because the compiler expected the types used for the
`MaxAmount` constant when configuring pallets in the runtime should implement
the `Get`  trait.

```rust
#[pallet::config]
pub trait Config: frame_system::Config {
 /// --- snip ---
 type MaxAmount: Get<u64>;
 }
```

But instead, a bare constant was found. A quick fix would have been to create a
 `MaxAmount` type and manually implement the `Get` trait for this type:

```rust
// Create a struct to act as a wrapper
struct MaxAmount;

// Implement the Get trait for the wrapper
impl Get<u32> for MaxAmount {
    fn get() -> u32 {
        MAX_AMOUNT
    }
}
```

However, this is where the `parameter_types!` macro comes in. This macro not
only automatically implements the `Get` trait for your constants but also
ensures that these constants are properly integrated into the chain's
metadata. This makes them easily accessible and modifiable through on-chain
governance. In-depth information about the `parameter_types!` macro can be
found in the going in-depth section.

## Solving the error

To solve the error,

- Navigate to the [r](http://lib.rs)untime’s [lib.rs](http://lib.rs) file

```bash
cd runtime/src/lib.rs
```

- Create the `parameter_types!` macro and move `const MaxAmount: u64 = 1_000_000;` inside the macro

```bash
parameter_types! {
 pub const MaxAmount: u64 = 1_000_000;
}
```

- Compile the node

```bash
cargo build --release
```

## Going in-depth

The `parameter_types!` macro is a Rust procedural macro in Substrate that
allows you to define constants for your runtime logic. At its core, this
macro automatically implements the `Get` trait for your constants. This allows
these constant values to be fetched efficiently across the entire runtime,
wherever the `Get` trait is implemented or expected.

```rust
parameter_types! {
    pub const BlockHashCount: u64 = 250;
    pub const MaximumBlockWeight: u32 = 1024;
}
```

In standard Rust, constants are usually defined with the `const` keyword. But
in Substrate, using `parameter_types!` provides type-safety benefits. It makes
sure that the constants are compatible with the types expected in the pallet
configuration traits.

One of the most compelling features of the `parameter_types!` macro is its
seamless integration with the chain's metadata. This makes these constants not
only easily accessible but also easily modifiable. That is, the values of
these constants can be updated through a decentralized voting mechanism,
avoiding the need for manual code changes.

Why is type safety so crucial here? In most languages, the type of a constant
is determined solely by its initial value. However, Substrate often requires
your constants to interact with generic types in pallet traits. The
`parameter_types!` macro ensures that your constants fulfill these trait
bounds, thus providing type safety.

```rust
#[pallet::config]
 pub trait Config: frame_system::Config {
  type MaxAmount: Get<u64>;
 }
```

In this case, using `parameter_types!` ensures that the constant will meet the
required trait bound of `Get<u32>`.

When a constant is used in multiple locations across a pallet, the `Get` trait
fetches it as a reference, saving memory and computational power. This
performance optimization, though subtle, can add up in a blockchain runtime
where efficiency

Errors in Substrate are not just mere annoyances; they can sometimes be
enigmatic. One such common error occurs when constants are not defined
within the `parameter_types!` macro.

Here's a simplified example of a typical error message:

```bash
error[E0573]: expected type, found constant `MaxAmount`
```

What this is telling you is that the Rust compiler expected a type that
implements the `Get` trait, but found a raw constant instead. Without using
`parameter_types!`, the constant hasn't been wrapped with a type that
implements the `Get` trait, leading to compatibility issues.

This leads to a more serious error:

```bash
error[E0152]: duplicate lang item in crate `std`: `panic_impl`.
```

This error indicates a more systemic issue related to linking multiple standard
libraries, which can happen due to incorrect constant configuration.

## Summary

The article serves as a hands-on tutorial for dealing with constants in
Substrate runtimes, highlighting the advantages of using the
**`parameter_types!`** macro.

This macro streamlines the process of declaring constants by automatically
implementing the **`Get`** trait, providing type safety, and allowing for easy
accessibility and modification. The tutorial illustrates this by showing how
defining pallet constants outside the **`parameter_types!`** macro can lead to
compile-time errors and even more complex issues related to system libraries.

To learn more about the concepts discussed in this guide, here are some
resources that we recommend:

- [Modify the runtime](https://docs.substrate.io/quick-start/modify-the-runtime/)
- [Runtime constants](https://docs.substrate.io/reference/how-to-guides/basics/configure-runtime-constants/)

To help us measure our progress and improve Substrate in Bits content, please
fill out our living [feedback form](https://airtable.com/shr7CrrZ5zqlhWEUD).
It will only take 2 minutes of your time. Thank you!
