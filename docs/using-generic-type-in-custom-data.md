# Using Generic Types in Custom Data in a Substrate Pallet

When developing a custom state transition logic with substrate, you’ll often need to create a custom data structure to handle information about the blockchain state or to temporarily store data before it is processed. 
This guide will walk you through a common problem faced when trying to implement a custom data type that leverages substrate rich type system, as well as an in-depth explanation of relevant concepts.

>To help us measure our progress and improve Substrate in Bits content, 
please fill out our living [feedback form](https://airtable.com/appc45lFGS94WumrY/tblnuIR8lSd4TX7IR/viwqMQuAR6zSDn765?blocks=hide). 
It will only take 2 minutes of your time. Thank you!

## Reproducing errors

For the sake of this guide, we’ve created a custom blockchain and built a new pallet. The blockchain serves as an archive of books and has a pallet that allows users to upload a summary of a book and retrieve a book if it is already archived. Currently, the only functionality in this pallet is the `archive_book` method which ensures all relevant information about a book is stored on the blockchain.

### Environment and project setup.

To follow along with this tutorial, ensure that you have the Rust toolchain installed

- Visit [substrate official documentation](https://docs.substrate.io/install/) page for the installation processes.
- Clone the project [repository](https://github.com/cenwadike/book-archive-pallet).

```rust
git clone https://github.com/cenwadike/book-archive-pallet.git
```

- Navigate into the project’s directory.

```rust
cd book-archive-pallet
```

- Run the command below to compile the node.

```rust
cargo build --release
```

While attempting to compile the node above, you’ll encounter an error similar to the one below: 

    error[E0412]: cannot find type `AccountId` in this scope
     --> src/lib.rs:72:15
     |
    72 |         BookSummary<AccountId, BlockNumber>,
    |                     ^^^^^^^^^ not found in this scope


and 
    
    error[E0412]: cannot find type `BlockNumber` in this scope
    --> src/lib.rs:72:26
     |
    72   |         BookSummary<AccountId, BlockNumber>,
     |                                ^^^^^^^^^^^


## Solving the error

The pallet has a unique data type (ie, BookSummary ) which was defined like so:

```rust
#[derive(Clone, Encode, Decode, Default, Eq, PartialEq, RuntimeDebug, TypeInfo)]
        pub struct BookSummary<AccountId, BlockNumber> {
                title: Vec<u8>,          // title of book
                author: Vec<u8>,         // author of book
                url: Vec<u8>,            // web url to off-chain storage
                archiver: AccountId,     // account id of archiver
                timestamp: BlockNumber,  // time when book was archived
        }
```

`BookSummary` has five components:

- **`title`**: The title of a book that is archived,  stored as a vector of `u8`.
- **`author`**: The author of a book that is archived, stored as a vector of `u8`.
- **`url`**: The web url of a book that is archived, stored as a vector of `u8`.
- **`archiver`**: The user who archived a book, stored as a vector of `AccountId`.
- **`timestamp`**: The block height when a user archived a book, stored as `BlockNumber`.

`BoookSummary` uses two FRAME generic types:
1. `AccountId`: which is used to record who archived a book.
2. `BlockNumber`: which is used to record the time when a book was archived.
The function body does the following:

The error we’re getting above originates from line 72:

```rust
#[pallet::storage]
pub(super) type ArchiveStore<T: Config> = StorageMap<
        _,
        Blake2_128Concat,
        T::Hash,
        BookSummary<AccountId, BlockNumber>,
        OptionQuery,
    >;
```

Rust compiler could not know where the type `AccountId` and `BlockNumber` within the scope of the custom data type `ArchiveStore`. It requires that `AccountId` and `BlockNumber` types are brought into the scope of our custom struct.

Because `AccountId` and `BlockNumber` are defined by default in our pallet `Config` trait, we can use them within the scope of our custom data type.

To resolve the error, replace 
```rust
#[pallet::storage]
pub(super) type ArchiveStore<T: Config> = StorageMap<
        _,
        Blake2_128Concat,
        T::Hash,
        BookSummary<T::AccountId, T::BlockNumber>,
        OptionQuery,
    >;
```
Take note of `T::AccountId` and `T::BlockNumber` which replace `AccountId` and `BlockNumber` respectively.

- Re-compile the node    
```rust
cargo build --release
```

## Going In-depth

### Inspecting the FRAME System `Config` trait
The System pallet defines the core data types used in a Substrate runtime. It also provides several utility functions and types for other pallets including our custom pallet.

The FRAME system `Config` is a system configuration trait that can be implemented for any pallet by a runtime. This means that a pallet can define a specific configuration that summarises all relevant substrate primitive data types required to fully construct the pallet and interact with its state and functions.

Any runtime that is constructed with a pallet configuration and implements a concrete `Config` trait of a pallet can utilize all the state and functions of that pallet.

To learn more about the types available in FRAME System `Config` visit the [documentation](https://paritytech.github.io/substrate/master/frame_system/pallet/trait.Config.html).


### A look at complementary errors you may encounter
Most of the common errors associated with using custom types are related to absence trait implemtation.

1. `MaxEncodedLen` is not implemented for `Vec<u8>`

A common error when using `Vec<u8>` is shown below:
```bash
trait `MaxEncodedLen` is not implemented for `Vec<u8>`
```
This error originates because the Rust compiler does not know the maximum length of the `Vec<u8>` at compile time.

A simple fix is to ensure that your `Pallet` struct has this `#[pallet::without_storage_info]` implemented like so:

```rust
#[pallet::pallet]
#[pallet::without_storage_info]
pub struct Pallet<T>(_);
```


2. trait bound `TypeInfo` is not satisfied; `TypeInfo` is not implemented for `custom_struct`.

Note that `TypeInfo` could be any other trait that is implemented using a derived macro.

The fix is frequently to add the derived macros to your custom struct like so:

```rust
#[derive(Clone, Encode, Decode, Default, Eq, PartialEq, RuntimeDebug, TypeInfo)]
pub struct BookSummary<AccountId, BlockNumber> {
    title: Vec<u8>,          // title of book
    author: Vec<u8>,         // author of book
    url: Vec<u8>,            // web url to off-chain storage
    archiver: AccountId,     // account id of archiver
    timestamp: BlockNumber,  // time when book was archived
}
```

3. `OptionQuery` vs `ValueQuery` vs `ResultQuery`

- `OptionQuery` should be your preferred when defining custom data without a default implementation. A query from such data, will either `Some` value if storage contains a value or `None` if there's no value in storage.

- `ValueQuery` always returns a value from storage or causes the program to panic loudly. You can use ValueQuery to return the default value if you have configured a specific default for a storage item or return the value configured with the OnEmpty generic.

- ResultQuery queries a result value from storage and returns an error if there's no value in storage.


## Summary

In this guide, you:

- Encountered a compilation error in the template due to not **`Config`** type being out of the scope of a custom data type.
- To fix the error, you replaced the problematic line of code with the appropriate line that include the generic type parameter **`<T>`**.

Also, you learned that:

- All Substrate pallets must define trait **`T`** which defines a Pallet configuration.
- This trait must implement FRAME system **`Config`** trait.
- A concrete Pallet trait **`Config`** is implemented on a runtime.
- A custom struct must implement all relevant traits.

To learn more about the concepts discussed in this guide, here are some resources that we recommend:

- [FRAME System](https://paritytech.github.io/substrate/master/frame_system/index.html/)
- [Runtime storage structures](https://docs.substrate.io/build/runtime-storage/#Querying-Storage/)

We’re inviting you to fill out our living [feedback form](https://airtable.com/shr7CrrZ5zqlhWEUD) to help us measure our progress and improve Substrate in Bits content. It will only take 2 minutes of your time. Thank you!

