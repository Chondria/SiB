---
tags:
  - substrate
keywords: [polkadot, substrate, node, std, runtime]
description: Build a substrate node from scratch (sub-series) part 1
updated: 2023-07-16
author: abdbee
duration: 3h
level: intermediate
---

# Build a substrate node from scratch (sub-series) part 1

The substrate node template goes a long way in helping substrate engineers compile their first node to begin their development journey as soon as possible. This is especially helpful when the focus is on understanding how to build the logic of the chain and not on other layers. However, one would quickly realise that there’s more than the node template. To truly be able to harness the features of Substrate, one needs to understand the inner workings of a node and how the various layers come together to make up a functioning node.  In this “Substrate-In-Bit” sub-series, We’ll break down a substrate node and teach you how to build your node from scratch.

For this sub-series, we’ll be building **Lancr**, a substrate-based freelance marketplace.

By the end of this first part of the sub-series, you should:

- Have a good understanding of the architecture of the blockchain and Substrate,
- Understand the usage of Substrate's core libraries,
- Have set up a new Rust project for your blockchain,
- Have configured the project’s runtime to be able to compile to WebAssembly (WASM).

**You can check out this project's repository [here](https://github.com/abdbee/Lancr)**


> We’re inviting you to [fill out our living feedback](https://airtable.com/shr7CrrZ5zqlhWEUD) form to help us measure our progress and improve Substrate in Bits content. It will only take 2 minutes of your time. Thank you!

## Environment set up.

To follow along with this tutorial, make sure you have *Rust toolchain* installed. Check out the official Substrate documentation for a [detailed guide](https://docs.substrate.io/install/rust-toolchain/) to installing the toolchain in your pc.

## A general look at blockchain architecture

A blockchain needs users to be able to submit transactions. Unexecuted transactions are stored in a transactions pool and the blockchains have logic that takes these transactions, makes checks, and updates storage based on the state changes observed by these transactions.

These changes are used to update the blockchain's database which keeps track of the current state of the blockchain. Each blockchain node is responsible for grouping transactions into blocks, processing the blocks with the transactions contained in them and updating the node’s database with the changes arising from the transactions’ execution.

Apart from being able to validate/execute transactions and updating the blockchain’s database with recent states, each blockchain node also needs to be able to “talk” with other blockchain nodes. And since each node can create a block, all the nodes need a unified way to reach an agreement as to which node will be producing the next block, and how the validity of this block will be verified by other nodes. 

Once a validating node has verified a particular block, the block is imported unto their node and the node’s database is updated with the states in the imported block. This components of the blockchain also need to be able to communicate with each other and the outside world.

## Substrate architecture

The logic that checks transactions and updates the blockchain's state is housed within the Substrate **Runtime.** Every other functionality required for nodes to function properly (eg reaching consensus, databases, node communications etc) are housed in the Substrate **core client**. The core client provides all the services needed for the blockchain to work as intended. 

However, although these two substrate components are distinguished in terms of functionality, they are highly dependent on each other and need to be able to communicate in order to perform operations. For example, the runtime needs to be able to fetch the recent nonce for a particular account from the key-value database housed in the core client, so it can increment the account’s nonce each time it makes a transaction. 

Communications between the Runtime and core client are facilitated by these [Runtime Apis](https://docs.substrate.io/reference/runtime-apis/).

![Substrate architecture](https://github.com/Chondria/SiB/blob/main/Images/Untitled%20Diagram.drawio.png)
### Substrate core libraries

In most cases, the core client uses the core client library to handle most of its services. On the other hand, the runtime has a rich FRAME library which contains logic that can be configured to suit the use-cases of the runtime. 

Both the *core client* library and the *FRAME* library leverage the interface, data structures and functions defined in the *Primitive libraries* to facilitate seamless communication and interaction between the on-chain (Runtime) and off-chain (Client) components of the node. This allows for effective transaction processing, state management, consensus mechanism operations, and other fundamental aspects of blockchain functioning.

Essentially, Primitives ensure that the runtime and client are "speaking the same language”. It serve as a common foundation for both the runtime and client, providing the necessary tools and structures for them to interact effectively. 

You can check out the libraries with the links below

- [Core client library](https://github.com/paritytech/substrate/tree/master/client)
- [FRAME library](https://github.com/paritytech/substrate/tree/master/frame)
- [Primitives](https://github.com/paritytech/substrate/tree/master/primitives)

## Set up a new rust project for your blockchain

Create a new project for your blockchain node by running the code below:

```
cargo new --bin lancr
cd lancr
```

For the skeleton node, we’ll be building a bare functionality for the main node (client) and the runtime. 

Let’s create a new rust package for the client within the lancr directory

```
cargo new --lib node
```

For the runtime, all runtimes will be housed in the `runtime` folder. We’re going to start with just one runtime called `prelancr` where we’ll launch and test all new features. Think of this as our `canary` runtime, similar to `Kusama`.

Run the commands below to create the runtime directory and a new runtime package called `prelancr`.

```
mkdir runtime
cd runtime
cargo new --lib prelancr
```

Now that we have the two major packages needed for our blockchain to run, we need to make sure that we can build and test both packages at once and that dependencies can be shared across both packages. It will also be good for both packages to share the same output (`target`) directory. To achieve all these, we include both packages in the same workspace in the project’s cargo.toml file

To do this, make sure you’re in the root directory of `lancr`. 

- Open the cargo.toml file
- Delete the dummy code in the file
- Include the code below:

```
[workspace]
members = [
    "node",
    "runtime/prelancr",
]
```

By doing this, both the node and `prelancr` runtime packages are now in the same workspace and they will share the same `Cargo.lock` file and `target` directory. This means that they will use the same version of each dependency and their compiled outputs will be stored in the same location. Running commands such as `cargo build` or `cargo test` from the workspace root will now apply to both the `node` and `runtime` packages. This facilitates consistency, dependency management, and eases the build process across both packages.

Also, include the code below in your cargo.toml file

```
[profile.release]
panic = "unwind"
```

This ensures that in the event of a serious error (a "panic") when your project is in 'release' mode, the program will not immediately stop, but will instead carefully clean up and check each variable ("unwind" the stack). This allows for a more thorough investigation of what went wrong, which can be especially useful for debugging.

Now compile the project to ensure that everything is in order:

```
cargo build --release
```

## Let’s ensure our runtime compiles to WebAssembly

Before we write the runtime’s logic, we need to make sure that in addition to native compilation of our runtime, it also compiles to WebAssembly (WASM). The WASM binary is included in on-chain storage. Including this binary on-chain helps prevent forks, as it ensures that all validators always execute the same version of the runtime (since validators can only run their native runtimes if its version is the same as that of the on-chain WASM runtime)

To ensure that your runtime can compile into WASM:

- Add `substrate_wasm_builder` as a dependency in the `prelancr` runtime (`lancr/runtime/prelancr`). This tool is designed for compiling substrate code to WASM.

```rust
[build-dependencies]
substrate-wasm-builder = { version = "5.0.0-dev", git = "https://github.com/paritytech/substrate.git", optional = true , branch = "polkadot-v0.9.42" }
```

- Enable the `std` feature for `substrate-wasm-builder`

```rust
[features]
default = ["std"]
std = [
    "substrate-wasm-builder"
]
```

- Create a new [build.rs](http://build.rs) file within the `prelancr` runtime (`lancr/runtime/prelancr`) and add the code below:

```rust

#[cfg(feature = "std")]
fn main() {
	substrate_wasm_builder::WasmBuilder::new()
		.with_current_project()
		.export_heap_base()
		.import_memory()
		.build()
}
```

***A note on `std` features***

Writing code for std and no_std  environments can be confusing at first, but here’re some points to keep in mind. 

- By default, all Rust code has access to the standard library (**`std`**). You can opt out of this by specifying **`#![no_std]`** at the top of your code, which signals that your code is intended for environments without the standard library.
- Enabling the `std` feature for a crate means the crate should be compiled with access to rust’s standard library. But not all environments have access to the standard library (eg., embedded systems, kernel development, or when targeting WebAssembly). Therefore, to ensure your code can function in these environments, you can use conditional compilation attributes like `#[cfg(feature = "std")]` and `#[cfg(not(feature = "std"))]` to differentiate parts of your code that require the standard library from those that don't.
- The `#[cfg(feature = "std")]` attribute is included for code blocks that require the standard library to compile and function properly. All code blocks with this attribute are only compiled when the std feature is enabled for the crate.
- If the std feature is disabled (eg, when compiling for no_std environments, the code blocks with the `#[cfg(feature = "std")]` attribute will be ignored.
- If you want a code block to be compiled only when the standard library is unavailable (ie when the std feature is disabled), the `#[cfg(not(feature = "std"))]` will be used.

## Summary

In this first part of the "node from scratch" sub-series, you were introduced to the process of setting up your environment for the development of a new Substrate blockchain.

First, we started by providing an overview of the general architecture of a blockchain and Substrate. Here, we highlighted the importance of the runtime and core client components in facilitating transactions, managing the state of the blockchain, enabling node-to-node communication, and more.

Next, we delved into the crucial role of Substrate's core libraries (core client library, FRAME library, and primitives) in facilitating communication and interaction between on-chain (runtime) and off-chain (client) components.

After this conceptual understanding, we proceeded to set up a new Rust project for our blockchain, named `lancr`. This involved creating a project workspace, and initiating the main node (client) and the runtime. We ensured that both the node and runtime packages share the same workspace to facilitate consistency, dependency management, and ease of building across both packages.

Next, we added important configurations in our `cargo.toml` file to ensure that our program doesn't immediately stop in the event of a serious error, but rather carefully cleans up and checks each variable. This configuration is crucial for efficient debugging.

Lastly, we took steps to ensure our runtime compiles into WebAssembly, by adding `substrate_wasm_builder` as a dependency and enabling the 'std' feature for it.

This first part has thus laid the foundation for building a Substrate node from scratch. In the following parts, we will build on this foundation, developing the logic for our node and refining its functionalities.

> We’re inviting you to [fill out our living feedback](https://airtable.com/shr7CrrZ5zqlhWEUD) form to help us measure our progress and improve Substrate in Bits content. It will only take 2 minutes of your time. Thank you!
