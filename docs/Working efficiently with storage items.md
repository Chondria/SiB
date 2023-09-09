---
tags:
  - substrate
keywords: [polkadot, substrate, node, storage, runtime]
description: Working efficiently with storage items
updated: 2023-09-03
author: abdbee
duration: 3h
level: intermediate
---


# Working efficiently with storage items

While Substrate’s architecture offers a developer-friendly abstraction (API)
layer for storage, making it simpler to persist data, efficient storage design
is crucial for performance, upgradeability, and resource utilization. This
guide takes a problem-based approach to common issues and best practices for
working with runtime storage items.

By the conclusion of this guide, developers will be equipped with a good
understanding of storage management, including but not limited to error
handling, data validation through the `ensure!` macro, and complex conditional
storage checks.

> To help us measure our progress and improve Substrate in Bits content, please
fill out our living [feedback form](https://airtable.com/shr7CrrZ5zqlhWEUD).
It will only take 2 minutes of your time. Thank you!

## Reproducing errors

### Environment and project setup

To follow along with this tutorial, ensure that you have the Rust toolchain
installed.

- Visit the [substrate official documentation](https://docs.substrate.io/install/)
page for the installation processes.

- Clone the project [repository](https://github.com/abdbee/sibchain/tree/writing-efficient-storage-items)

```
git clone -b writing-efficient-storage-items --single-branch https://github.com/abdbee/sibchain.git
```

- navigate to the project’s directory

```
cd sibchain

```

- Run the command below to compile the node

```
cargo build --release

```

- Run your node

```rust
./target/release/node-template --dev
```

- Visit Polkadotjs to connect to the local node:
  - Go to the **development** tab
  - Select **local node**
  - Click on the **switch** button

- Once the node’s UI is up, go to the extrinsic page and call the
`templateModule.setMemberId(memberId)` extrinsic using Alice’s account with the
following parameters:
  - MemberId: 1000

- Submit the transaction
  
- Now repeat the steps above for Bob’s account.
(i.e., set the memberId to 1000 too).

You’ll immediately notice that there’s a problem. Multiple accounts can have
the same member IDs, which is meant to be unique for each account. If you query
the `templateModule.idOf` storage item, you’ll notice that multiple accounts
have the same member ID of 1000.

- Now try changing the memberId for Alice to `500` using the
`templateModule.setMemberId(memberId)` extrinsic. You’ll notice that the change
goes through, which isn’t the intention of setting unique member IDs
(the intention is for IDs to be set once and remain unchangeable)

## Solving the problem

To solve this problem, we need to introduce checks that make sure that a
member cannot claim an ID that has already been taken and also to make sure
that an account can’t change its ID once it has been set.

To do this:

- Create a new storage item that keeps track of all used member IDs

```rust
  //mapping each u32 member ID to a bool. When an ID is taken, the value will be set to true
 #[pallet::storage]
 #[pallet::getter(fn taken_memberids)]
 pub(super) type TakenMemberIds<T: Config> = StorageMap<_, Twox64Concat, u32, bool, ValueQuery>;
```

- Write the errors that will be printed when:
  - An account has already claimed a member ID
  - A particular Member ID has been taken

    ```rust
      #[pallet::error]
     pub enum Error<T> {
      //---snip-----
    
      /// AccountId already exists
      AccountIdExists, //<--- Add this line
    
      /// Member Id taken
      MemberIdTaken, //<--- Add this line
    
     }
    ```

- Use the ensure! macro in the `set_member_id` dispatchable to :
  - Check if an account ID already exists in storage
  (in which case it will return the `AccountIdExists` error).
  - Check if the member ID already exists for any account
  (in which case it will return the `MemberIdTaken` error)
  - In addition to the above, mark the Member ID taken by adding the ID to the
  `TakenMemberIds` storage item and setting its value to `true`:

    ```rust
        #[pallet::call_index(1)]
        #[pallet::weight(100_000_000)]
      pub fn set_member_id (origin: OriginFor<T>, member_id: u32) -> DispatchResult {
         //---snip-----
        
       /// Add the lines below:
    
          // Check if accountId already exists in storage
       ensure!(!<IdOf<T>>::contains_key(&who), Error::<T>::AccountIdExists);
    
       // Check if the member ID already exists for any account
       ensure!(!TakenMemberIds::<T>::contains_key(member_id), Error::<T>::MemberIdTaken);
    
       // Mark this member ID as taken
       TakenMemberIds::<T>::insert(member_id, true);
    ```

- Compile and run your node:

    ```rust
    cargo build --release
    
    ./target/release/node-template --dev
    ```

You’ll now notice that the chain returns an error when you try to change the ID
of an account or reassign an already-taken member ID to a different account.

## Going in-depth

Substrate offers three primary types of storage items:

1. **Storage Value**: Stores single data items. Ideal for system-wide configurations or flags.
2. **Storage Map**: A Key-Value storage, often used for storing records indexed by account ID.
3. **Storage Double Map**: Allows a two-key to value mapping. Suitable for complex nested data structures.

### **Storage Value**

Used for global configurations. Avoid using Storage Value for user-specific
data to prevent excessive storage reads, which can impact performance.

### **Storage Map**

This is your go-to for any account-specific data. Always ensure to map to the
least amount of data necessary. For example, rather than storing a complex
struct, store keys to the struct and keep the elements in a separate map.

### **Storage Double Map**

Useful for multi-dimensional data. Be wary of the computational complexity it
can introduce. Always have clear documentation about the two keys to avoid
misuse.

### Things to take note of when declaring storage items

- Always ensure that keys are unique. Test rigorously to ensure no key
overlaps. The use of non-unique keys in storage maps can lead to data being
overwritten.
- Optimize your business logic to minimize storage operations. Frequent storage
operations can lead to increased computational costs.
- Always validate data at the runtime level using methods like `ensure!` to
enforce business logic constraints. Not validating data before storing can lead
to data integrity issues.

### ****Best Practices for Storage Checks****

Implementing storage checks usually involves using conditional statements or
macros like `ensure!` in Rust to validate data or state conditions before
proceeding with storage operations.

```rust
ensure!(!IdOf::<T>::contains_key(&who), Error::<T>::AccountIdExists);

```

In this example, the `ensure!` macro checks whether an account ID is already
present in the storage map. If it is, the function returns an error, avoiding
any unintended overwrites and thereby preserving data integrity.

- **Using Timeouts or Block Heights**

    When dealing with actions that are time-sensitive, it's often necessary to
    incorporate timeouts or specific block heights as a condition.

    ```rust
    rustCopy code
    ensure!(<BlockNumber<T>>::get() <= expiry_block, Error::<T>::ActionExpired);
    
    ```

    In this case, the operation will only proceed if the current block number
    is less than or equal to the `expiry_block`, failing which, an error will
    be thrown.

- **Leveraging Enum States**

    Sometimes, data can have multiple states. Using enums for these states and
    checking against them can be a prudent storage check.

    ```rust
    ensure!(<ContractState<T>>::get(&contract_id) == ContractState::Active, Error::<T>::ContractNotActive);
    
    ```

    Here, the contract will proceed only if its state is `Active`.

- **Composite Checks**

    Sometimes, complex business logic necessitates multiple checks bundled into
    a single composite condition.

    ```rust
    rustCopy code
    ensure!(<IsUserVerified<T>>::get(&user) && <HasSufficientBalance<T>>::get(&user), Error::<T>::CannotProceed);
    
    ```

    Here, the action proceeds only if the user is verified and has sufficient balance.

- **Logical Checks**

    Not all conditions will be straightforward true/false checks. Sometimes,
    logic gates (`AND`, `OR`, `NOT`) can make these conditions more effective.

    ```rust
    rustCopy code
    ensure!(!(condition_A && condition_B) || condition_C, Error::<T>::InvalidCondition);
    
    ```

### **Case Study: Ensuring Uniqueness of Member IDs**

Here’s a practical example of a storage check to ensure that a member ID is
unique across accounts:

```rust
rustCopy code
ensure!(!TakenMemberIds::<T>::contains_key(member_id), Error::<T>::MemberIdTaken);

```

Before allowing the extrinsic to set a member ID, it checks a separate
`TakenMemberIds` storage map to ensure that the ID is unique. If it’s not,
the extrinsic fails, protecting against duplicate IDs and ensuring each member
has a unique identifier.

## Conclusion

In this "Substrate in Bits" content, we have explored the challenges and
solutions surrounding storage management, and best practices when dealing with
storage items. Starting with a hands-on tutorial, we how multiple accounts can
erroneously end up with the same member IDs. The tutorial then proceeds to
introduce fixes like creating a storage item to track taken member IDs and
enforcing checks to prevent ID duplication and modification.

We’ve also shed light on how to enforce data integrity and state conditions
using the `ensure!` macro and provides several other techniques for robust
storage checks. These include the use of timeouts or block heights for
time-sensitive actions, leveraging Enum States for data that can exist in
multiple states, and composite checks for complex business logic.

To learn more about the concepts discussed in this guide, here are some
resources that we recommend:

- [Runtime Storage](https://docs.substrate.io/build/runtime-storage/)
- [Custom pallets](https://docs.substrate.io/build/custom-pallets/)
- [Create a storage structure](https://docs.substrate.io/reference/how-to-guides/pallet-design/create-a-storage-structure/)

> To help us measure our progress and improve Substrate in Bits content,
please fill out our living [feedback form](https://airtable.com/shr7CrrZ5zqlhWEUD).
It will only take 2 minutes of your time. Thank you!
