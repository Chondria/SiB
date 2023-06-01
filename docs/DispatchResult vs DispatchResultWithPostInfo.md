---
tags:
  - substrate
keywords: [polkadot, substrate, error handling, dispatchables, pallets]
description: DispatchResult vs DispatchResultWithPostInfo
updated: 2023-05-21
author: abdbee
duration: 3h
level: intermediate
---

# DispatchResult vs DispatchResultWithPostInfo

When writing functions for substrate modules, you generally have the option to return their values as one of two result types: `DispatchResult` or `DispatchResultWithPostInfo` . You can create your own custom types and use them as return types, of course, provided they implement the necessary traits. 

In practice, however, using `DispatchResult` and `DispatchResultWithPostInfo` is recommended because they provide a consistent interface and are integrated with Substrate's error handling mechanism.  

In this Substrate in Bits content, we will take a problem-based approach to knowing the difference between these two common return types.

> To help us measure our progress and improve Substrate in Bits content, please fill our living [feedback form]([https://airtable.com/appc45lFGS94WumrY/tblnuIR8lSd4TX7IR/viwqMQuAR6zSDn765?blocks=hide](https://airtable.com/appc45lFGS94WumrY/tblnuIR8lSd4TX7IR/viwqMQuAR6zSDn765?blocks=hide). It will only take 2 minutes of your time. Thank you!

## Reproducing errors

To reproduce the errors for this project you’ll have to set up your environment and clone the repository of the node for this project.

## **Environment and project set up.**

To follow along with this tutorial, ensure that you have the Rust toolchain installed.

- Visit the [substrate official documentation](https://docs.substrate.io/install/) page for the installation processes.

- Clone this project’s [repository]([https://github.com/abdbee/DispatchResult-vs-DispatchResultWithPostInfo](https://github.com/abdbee/DispatchResult-vs-DispatchResultWithPostInfo).

```rust
git clone https://github.com/abdbee/DispatchResult-vs-DispatchResultWithPostInfo.git
```

- Navigate into the project’s directory.

```
cd DispatchResult-vs-DispatchResultWithPostInfo
```

- Run the command below to compile the node.

```
cargo build --release
```

While attempting to compile the node above, you’ll encounter an error similar to the one below:

```
error[E0308]: mismatched types
   --> pallets/template/src/lib.rs:120:7
    |
120 |             Ok(())
    |                ^^ expected struct `PostDispatchInfo`, found `()`

For more information about this error, try `rustc --explain E0308`.
error: could not compile `pallet-template` due to previous error
warning: build failed, waiting for other jobs to finish...
error: build failed
```

## Solving the error

Let’s look at the function that this error arises from:

```rust
#[pallet::weight(10_000 + T::DbWeight::get().writes(1))]
pub fn add_number(origin: OriginFor<T>, number: u32) -> DispatchResultWithPostInfo {
    // Ensure the call is signed
    let _ = ensure_signed(origin)?;

    // Validate the input
    ensure!(number >= 100, Error::<T>::NumberTooLow);  // Returns an error if number is less than 100
    ensure!(number <= 1000, Error::<T>::NumberTooHigh); // Returns an error if number is greater than 1000

    // Store the number in the storage
    <Numbers<T>>::put(number);

    // Return success without additional info
    Ok(())
}

```

The function above was declared to return a `DispatchResultWithPostInfo` type. But at the end of the function, it’s returning `Ok(())`, which is of the `DispatchResult` type.

To correct this, you need to ensure that the returned value is of the `DispatchResultWithPostInfo` type. you can convert a `DispatchResult` into a `DispatchResultWithPostInfo` by calling `.into()` on the `DispatchResult` type. This will automatically create a `PostDispatchInfo` with default values (`actual_weight: None, pays_fee: Pays::Yes`).

- Replace `Ok(())` in the function above with the following code:

```rust
Ok(().into())
```

- Re-compile the node

```rust
cargo build --release
```

You can also specify the actual values for the fields of `PostDispatchInfo` manually. For example, you can specify the actual weight to be 6_500. If `actual_weight` is provided, the value you specified will be used instead of the weight originally declared with the `#[pallet::weight]` attribute.

- Replace `Ok(().into())`  in the function above with the following code:

```rust
//Assume that the actual weight consumed is 6_500 units
let actual_weight = 6_500;
Ok(Some(actual_weight).into())
```

- Re-compile the node.

In this case, we are manually specifying `PostDispatchInfo` to say the `actual_weight` used was `6_500`. The runtime will use this value for weight calculation instead of the initially declared weight. `.into()` converts the `DispatchResult` type into a `DispatchResultWithPostInfo` with the explicitly defined `actual_weight` value and the default `pays_fee:` value (ie, `pays_fee: Pays::Yes`)

But if you don’t want to return any post-dispatch info, you can declare the function to return the `DispatchResult` type. In that case, the function can return  `Ok(())` without an error.

## Going in-depth

### DispatchResult

`DispatchResult` is a simple enumeration that indicates whether a dispatchable call was successful or failed. It can either be **`Ok`** (the call succeeded, and there's no value to return), or **`Err`** (the call failed, and here's why). You would use **`DispatchResult`** when you don't need to return any additional information about the call execution, such as post-dispatch weight adjustments.

```rust
pub type DispatchResult = Result<(), DispatchError>;
```

The `Err` variant (ie., `DispatchError`)is an enum that contains the possible reasons why a dispatch call might have failed.

[source](https://paritytech.github.io/substrate/master/sp_runtime/enum.DispatchError.html)

```rust
pub enum DispatchError {
    Other(&'static str),
    CannotLookup,
    BadOrigin,
    Module(ModuleError),
    ConsumerRemaining,
    NoProviders,
    TooManyConsumers,
    Token(TokenError),
    Arithmetic(ArithmeticError),
    Transactional(TransactionalError),
    Exhausted,
    Corruption,
    Unavailable,
    RootNotAllowed,
}
```

This enum represents all the possible errors that can occur during dispatch. When `DispatchResult` is used as the return type, Error types in a function have to be converted into the `DispatchError` type before they can be returned. For an error type in a function to be converted into the `DispatchError` type, `DispatchError` must implement the `From` trait that specifies how to convert that particular error into a `DispatchError`.

To understand this, let’s take a look at the function below:

```rust
#[pallet::call_index(4)]
pub fn transfer_all(
    origin: OriginFor<T>, 
    dest: AccountIdLookupOf<T>, 
    keep_alive: bool,
) -> DispatchResult {
    let transactor = ensure_signed(origin)?;
    
    let keep_alive = if keep_alive {
        Preserve 
    } else { 
        Expendable 
    };

    // --- snip ---

    Ok(())
}

```

In line 7, `ensure_signed(origin)?;` helps ensure the transaction is originating from a signed source. It returns a `Result` type that, if successful, contains the AccountId of the signer, but if unsuccessful, contains a `BadOrigin` error. You can check the source code of the `ensure_signed(o)` function [here]([https://paritytech.github.io/substrate/master/src/frame_system/lib.rs.html#918-926](https://paritytech.github.io/substrate/master/src/frame_system/lib.rs.html#918-926)

This `BadOrigin` error is represented by a struct in the [sp_runtime]([https://paritytech.github.io/substrate/master/src/sp_runtime/traits.rs.html#180](https://paritytech.github.io/substrate/master/src/sp_runtime/traits.rs.html#180) module.

```rust
pub struct BadOrigin;
```

But since this function is expected to return a `DispatchResult` type, it needs to convert this error type to a `DispatchError`. For this to be possible the **`From<BadOrigin>`** trait needs to be implemented for **`DispatchError`** .

```rust
// Implement the `From` trait for converting `BadOrigin` into `DispatchError`.
impl From<crate::traits::BadOrigin> for DispatchError {
    // Define the conversion function.
    fn from(_: crate::traits::BadOrigin) -> Self {
        // Convert `BadOrigin` into `DispatchError::BadOrigin`.
        Self::BadOrigin
    }
}
```

`From<BadOrigin>` allows for the automatic conversion of a `BadOrigin` type into a `DispatchError::BadOrigin` enum variant, and this is returned by the `?` operator if the `ensure_signed(origin)` function returns an error variant.

In essence,  `From<ErrorType>` trait has to be implemented for `DispatchError` for each error type that could be returned. You can check out the implementations for the error types [here]([https://paritytech.github.io/substrate/master/src/sp_runtime/lib.rs.html](https://paritytech.github.io/substrate/master/src/sp_runtime/lib.rs.html)

In summary, **`DispatchResult`** is used as the return type for functions that mutate state but don't need to return data. The purpose of the function is to make changes, and its success or failure is the only relevant information to return.

### DispatchResultWithPostInfo

`DispatchResultWithPostInfo` is a return type that is designed to provide additional information about the execution of a call beyond just success or failure. It allows developers to convey information about the computational weight consumed by a call and whether the call should pay fees.

Underlying this, `DispatchResultWithPostInfo` is a type alias for `DispatchResultWithInfo<PostDispatchInfo>.`

```

pub type DispatchResultWithPostInfo = DispatchResultWithInfo<PostDispatchInfo>;

```

`DispatchResultWithInfo<T>` is a more general type alias for `Result<T, DispatchErrorWithPostInfo<T>>`. This is designed to return either a success (with some type **`T`** of data) or an error (along with some type **`T`** of data).

```

pub type DispatchResultWithInfo<T> = Result<T, DispatchErrorWithPostInfo<T>>;

```

In the context of `DispatchResultWithPostInfo`, **`T`** is replaced with `PostDispatchInfo`.

`PostDispatchInfo` is a struct carrying additional data such as the actual computational weight consumed by the dispatchable call, and information about whether the call should pay fees.

```rust
pub struct PostDispatchInfo {
    pub actual_weight: Option<Weight>,
    pub pays_fee: Pays,
}
```

When a function returns a `DispatchResultWithPostInfo`, it could return an **`Ok(PostDispatchInfo)`** signifying success, or an **`Err(DispatchErrorWithPostInfo)`** indicating failure. If successful, **`PostDispatchInfo`** can carry additional data like the actual weight consumed. In the event of failure, **`DispatchErrorWithPostInfo`** contains the error along with additional **`PostDispatchInfo`**.

For instance, consider the following return statement:

```

Ok(PostDispatchInfo {
    actual_weight: Some(5000),
    pays_fee: Pays::Yes,
})
```

This indicates that the dispatchable function was successful, consumed 5000 units of weight, and that the caller should pay a fee.

On the other hand, an error return might look like this:

```
// Return an error along with additional post-dispatch information
Err(DispatchErrorWithPostInfo {
    // Specify post-dispatch information
    post_info: PostDispatchInfo {
        // Indicate the actual computational weight used by the function
        actual_weight: Some(3000),
        // Specify that the function does not pay a fee
        pays_fee: Pays::No,
    },
    // Indicate the specific error that occurred
    error: DispatchError::BadOrigin,
})
```

This signals that the function failed with a **`BadOrigin`** error, consumed 3000 units of weight by the time the error occurred, and the caller should not be charged a fee.

## Summary
This guide provides an in-depth exploration into the role of `DispatchResult` and `DispatchResultWithPostInfo` return types when writing functions for Substrate modules. 

The `DispatchResult` enumeration indicates the success or failure of a dispatchable call. `DispatchResultWithPostInfo`, on the other hand, provides additional post-dispatch information such as computational weight consumed by the call and fee payment details.

By using a problem-based approach, the guide highlights a typical error related to mismatched return types, specifically when a DispatchResult type is returned for a function declared to return DispatchResultWithPostInfo. Solutions are provided to correct the issue, demonstrating how to return PostDispatchInfo with default values or manually specified ones.

A further dissection of the `DispatchResult` and `DispatchResultWithPostInfo` types illuminates their underlying structure and interaction with the Substrate framework. It demonstrates how various errors are handled and converted into a DispatchError type, which forms a part of the DispatchResult.

For more resources on the concepts discussed in this article, check out the [official substrate documentation](https://docs.substrate.io/build/tx-weights-fees/).

> We’re inviting you to [fill out our living feedback](https://airtable.com/shr7CrrZ5zqlhWEUD) form to help us measure our progress and improve Substrate in Bits content. It will only take 2 minutes of your time. Thank you!
