---
tags:
  - substrate
keywords: [polkadot, substrate, error handling, macros, pallets, rust]
description: Let’s distill the #[pallet::storage] macro
updated: 2023-06-05
author: abdbee
duration: 3h
level: intermediate
---

# Let’s distill the #[pallet::storage] macro

The **`#[pallet::storage]`** macro is required for defining the state of a Substrate runtime module. It is utilized in the creation of pallet storage items. These items can be viewed as variables that the pallet can use to track the state between blocks which allows for the storage and retrieval of data.

In essence, **`#[pallet::storage]`** maps a specific type to a specific key in the storage system, providing an abstraction over the raw key-value storage that Substrate provides.

Let’s take an error-based approach to understand why this macro is needed and what happens when it’s not included when creating storage items.

To help us measure our progress and improve Substrate in Bits content, please fill out our living [feedback form](https://airtable.com/shr7CrrZ5zqlhWEUD). It will only take 2 minutes of your time. Thank you!

## Reproducing errors[](https://polkadot.study/#reproducing-errors)

### Environment and project setup[](https://polkadot.study/#environment-and-project-set-up)

To follow along with this tutorial, ensure that you have the Rust toolchain installed.

- Visit the [substrate official documentation](https://docs.substrate.io/install/) page for the installation processes.
- Clone the project [repository](https://github.com/abdbee/sibchain/tree/pallet_storage_macro).

```rust
git clone -b pallet_storage_macro --single-branch https://github.com/abdbee/sibchain.git
```

- navigate to the project’s directory

```rust
cd sibchain
```

- Run the command below to compile the node

```rust
cargo build --release
```

While attempting to compile the node above, you’ll encounter an error similar to the one below:

```
error: expected one of: `config`, `pallet`, `hooks`, `call`, `error`, `event`, `origin`, `inherent`, `storage`, `genesis_config`, `genesis_build`, `validate_unsigned`, `type_value`, `extra_constants`, `composite_enum`
  --> pallets/template/src/lib.rs:39:12
   |
39 |     #[pallet::getter(fn something)]
   |               ^^^^^^

error[E0432]: unresolved import `pallet`
 --> pallets/template/src/lib.rs:6:9
  |
6 | pub use pallet::*;
  |         ^^^^^^ help: a similar path exists: `frame_system::pallet`

For more information about this error, try `rustc --explain E0432`.
error: could not compile `pallet-template` due to 2 previous errors
warning: build failed, waiting for other jobs to finish...
error: failed to run custom build command for `node-template-runtime v4.0.0-dev (/workspace/NextSimpleStarter/sibchain/runtime)`

Caused by:
  process didn't exit successfully: `/workspace/NextSimpleStarter/sibchain/target/release/build/node-template-runtime-4ca8989960232eeb/build-script-build` (exit status: 1)
  --- stdout
```

This error is saying that the getter attribute requires another attribute to function. Without the **`storage`** attribute, the **`getter`** attribute is invalid because it's typically used to create a getter function for a storage item.

`#[pallet::storage]` is also needed to declare and define the storage items within the Substrate pallet. It provides an interface for manipulating data that the pallet stores in the chain's state. Without it, the storage items defined in the pallet would not be recognized by the Substrate runtime, and hence would not be accessible or modifiable. This annotation plays an important role in structuring the storage logic of the pallet and interfacing it with Substrate's runtime storage system.

Not including `#[pallet::storage]` when creating storage items essentially leaves them in a state of limbo within the codebase. In other words, the items are part of the code, but they are not recognized as part of the runtime state. Hence, they cannot be properly read, manipulated, or utilized in any meaningful way in the context of the blockchain's state.

## Solving the error

- Navigate to the pallet’s [lib.rs](http://lib.rs) file

```rust
cd pallets/template/src/lib.rs
```

- include the `#[pallet::storage]` macro before the storage item.

```rust
	#[pallet::storage] // <- add this line
	#[pallet::getter(fn something)] 
	pub type Something<T> = StorageValue<_, u32>;
```

This will flag the type as an on-chain storage item rather than a standard Rust variable. This annotation will allow the Substrate framework to automatically generate the necessary metadata, prefix structures, and getter methods that allow your storage item to interact with the blockchain state effectively and securely. It will also ensure the runtime and other components of your pallet can correctly identify and access this storage item. 

## Going in-depth

The **`#[pallet::storage]`** macro in Substrate is designed to provide a simplified way to define on-chain storage structures within a pallet. It's part of Substrate's larger pallet macro system, which helps abstract away much of the complexity of writing runtime code.

When you define a storage item within a pallet using the **`#[pallet::storage]`** macro, you're doing several things. Firstly, you're declaring a specific piece of on-chain storage. This isn't just a variable in your program; it's a piece of data that's stored on the blockchain, persistent across blocks, and accessible to all nodes in the network.

However, the macro does much more than just declare the storage. It is involved in the generation of necessary metadata for each storage item. This metadata is essential for runtime introspection and interaction with the storage via tools such as RPC calls. It includes information like the storage item's name, type, visibility, and associated documentation.

Moreover, for each storage item defined, the macro generates getter methods. These methods provide a controlled way to access the data stored in the storage items from other parts of your pallet code. This encapsulation ensures the integrity of the stored data and helps manage how the data is accessed or modified.

Another critical task the storage macro performs is creating a unique prefix for each storage item. These prefix structures are integral to generating unique keys for each storage item. These keys are crucial for storing and fetching data from the blockchain's state.

The Rust compiler and the Substrate framework rely on the **`#[pallet::storage]`** macro to correctly identify, process, and handle storage items in a pallet. Now, if you attempt to define a storage item without including the **`#[pallet::storage]`** macro, the Rust compiler won't recognize it as a storage item, and instead, treat it as a regular Rust variable. This could lead to a host of errors.

For instance, the macro system won't generate metadata for your storage item, and it will be missing from the blockchain's state metadata. This can result in runtime errors when you try to access the item via RPC calls.

Similarly, without the storage macro, no getter method is created. Thus, the rest of your pallet code won't have access to the storage data in a controlled manner, which could lead to data integrity issues.

Finally, without the storage macro, no unique prefix is generated for your storage item. This could lead to key collisions in the blockchain state, where multiple pieces of data are associated with the same key. As you can imagine, this can cause a variety of runtime issues, including data corruption.

## Summary

In this Substrate in Bits content, we explored the importance of the **`#[pallet::storage]`** macro in defining the state of a Substrate runtime module and managing on-chain storage. Essentially, this macro links a particular type to a specific key in the storage system, offering an elevated layer of abstraction over the raw key-value storage native to Substrate.

We delved into the consequences of omitting this macro during storage item creation, showcasing the errors and hurdles it poses. Without **`#[pallet::storage]`**, our storage items would remain unrecognized by the Substrate runtime system, rendering them inaccessible and unmodifiable. They would effectively be in a state of limbo within the codebase—part of the code, but not recognized as part of the runtime state.

Going deeper, we unveiled that the Storage Macro in Substrate offers a simplified route to defining on-chain storage structures. It is more than a declaration tool—it's instrumental in creating unique keys for storage items, generating getter methods, and crafting essential metadata.

If the **`#[pallet::storage]`** macro is not used, the Rust compiler treats the storage item as a standard Rust variable. This omission disrupts the generation of crucial metadata, creates access problems through missing getter methods, and can lead to key collisions due to absent unique prefixes. 

To learn more about the concepts discussed in this guide, here are some resources that we recommend:

- [FRAME Macros](https://docs.substrate.io/reference/frame-macros/)
- [Runtime Storage structures](https://docs.substrate.io/build/runtime-storage/)
- [The little book of Rust macros](https://danielkeep.github.io/tlborm/book/)

To help us measure our progress and improve Substrate in Bits content, please fill out our living [feedback form](https://airtable.com/appc45lFGS94WumrY/tblnuIR8lSd4TX7IR/viwqMQuAR6zSDn765?blocks=hide). It will only take 2 minutes of your time. Thank you!
