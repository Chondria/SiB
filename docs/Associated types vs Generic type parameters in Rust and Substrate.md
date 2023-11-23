---
tags:
  - substrate
keywords: [polkadot, substrate, associated types, generics, type parameters runtime]
description: Associated types vs Generic type parameters in Rust and Substrate
updated: 2023-11-23
author: abdulbee
duration: 3h
level: intermediate
---

## Introduction
It's important to understand the concepts of associated types and generic type parameters when working with Rust and Substrate. These features provide flexibility and reusability in code, allowing for clear associations between traits and their associated or generic types. In this guide, we go through what associated types and generic type parameters are, the difference between them, where/how they're used in Substrate, and points to keep in mind when using them.

> Help us measure our progress and improve Substrate in Bits content by filling
out our living [feedback form](https://airtable.com/shr7CrrZ5zqlhWEUD).
Thank you!

## Associated types
Associated types define types that are associated with a particular trait. The concrete types for these types are specified when implementing the trait for a type. Associated types are used where you expect that there will only be one implementation of the trait for a given type.

Here's an example:
```rust
pub trait Block {
        type Hash: Hash;
        type Transaction: Transaction;
        // Other associated types and methods...
    }
    
    pub trait Transaction {
        type Sender: Account;
        type Recipient: Account;
        // Other associated types and methods...
    }
    
    pub trait Account {
        // Methods and associated types for account management...
    }
    
    pub struct DeBlock;
    
    impl Block for DeBlock {
        type Hash = [u8; 32];
        type Transaction = MyTransaction;
        // Implementations for other associated types and methods...
    }
    
    pub struct DeTransaction;
    
    impl Transaction for DeTransaction {
        type Sender = MyAccount;
        type Recipient = MyAccount;
        // Implementations for other associated types and methods...
    }
    
    pub struct MyAccount;
    
    impl Account for MyAccount {
        // Implementations for account management methods and associated types...
    }
```
The example above has the definition of 3 traits:
    - The `Block` trait, which has the `Hash` and `Transaction` associated types
    - The `Transaction` trait which has the `Sender` and `Recipient` associated types
    - The `Account` trait for account management.

The associated types used here ensure that custom chains (ie, different implementations) are able to decide on their own definition of what each of these types should be. In the example above, the `Deblock`, `DeTransaction`, and `MyAccount` types were created and the `Block`, `Transaction`, and `Account` traits were implemented for them with the definition of concrete types. 

Associated types are suitable for the `Block`, `Transaction`, and `Account` traits because these traits represent concepts where it is not expected to have multiple instances of their associated types. 

For example, a blockchain typically has a single type that represents blocks. By using associated types, we can define these specific types directly within the trait, providing a clear and concise representation of the expected associations.

## Generic type parameters
Generic type parameters are a powerful feature that allows you to write generic code that can be reused with different types. They provide flexibility and enable you to write more generic and reusable functions, structs, and traits. Generic type parameters are used when it's plausible to have multiple implementations of the trait for a given type.

Here's an example:

```rust
pub trait Formatter<T> {
    fn format(&self, value: T) -> String;
}

pub struct StringFormatter;

impl Formatter<String> for StringFormatter {
    fn format(&self, value: String) -> String {
        value
    }
}

pub struct IntegerFormatter;

impl Formatter<i32> for IntegerFormatter {
    fn format(&self, value: i32) -> String {
        value.to_string()
    }
}

pub struct CustomType {
    // Implementation details...
}

pub struct CustomTypeFormatter;

impl Formatter<CustomType> for CustomTypeFormatter {
    fn format(&self, value: CustomType) -> String {
        // Custom formatting logic for CustomType
        // Implementation details...
    }
}
```

In this example, we have a `Formatter` trait with a generic type parameter T. The trait defines a single method format that takes a value of type T and returns a formatted string.

We then have multiple implementations of the Formatter trait for different types. The StringFormatter implements Formatter for the String type, allowing us to format strings. The IntegerFormatter implements Formatter for the i32 type, enabling us to format integers. Finally, the CustomTypeFormatter implements Formatter for a custom type CustomType, allowing us to define custom formatting logic specific to that type.

By using generic type parameters, we can have multiple implementations of the Formatter trait for different types. Each implementation can provide its own specific formatting logic tailored to the given type. This flexibility allows us to reuse the Formatter trait with various types and have different formatting behaviors based on the type being formatted.

In this example, the use of generic type parameters makes sense because we expect to have multiple implementations of the Formatter trait for different types, enabling us to format values in a type-specific manner.


Essentially, generic type parameters are more suitable when you anticipate having multiple implementations of a trait for different types. On the other hand, associated types are more suitable when you expect that there will only be one implementation of the trait for a given type.

## Where they are used in substrate

Let's take a look at the code snippets below:
```rust
#[pallet::pallet]
pub struct Pallet<T>(_);

#[pallet::config]
pub trait Config: frame_system::Config {
	/// Because this pallet emits events, it depends on the runtime's definition of an event.
	type RuntimeEvent: From<Event<Self>> + IsType<<Self as frame_system::Config>::RuntimeEvent>;
	/// Type representing the weight of this pallet
	type WeightInfo: WeightInfo;
	}

pub type MagicNumber<T> = StorageValue<_, u32>;

#[pallet::call]
impl<T: Config> Pallet<T> {
    #[pallet::call_index(0)]
	#[pallet::weight(T::WeightInfo::do_something())]
    pub fn set_magic_number(who: T::AccountId, value: u32) {
        //function body
    }
}

```
The `Pallet` struct is defined with the generic type parameter `T`, which represents the configuration trait of the runtime.
By using the generic type parameter `T`, the `Pallet` struct can be instantiated with different configurations of the Substrate runtime. This allows the storage item to work with different types of account IDs, depending on the specific configuration used. Notice that the `set_magic_number` has the generic type `T::AccountId` as a parameter, allowing the storage item to handle different types of account IDs based on the runtime configuration being used.

By representing the runtime as a generic type parameter `T`, the Pallet struct becomes adaptable and can be used with different runtimes that have varying configurations, as long as those runtimes implement the Config trait defined for the pallet.

The `Config` trait, as shown above, defines two associated types: `RuntimeEvent` and `WeightInfo`. These associated types are not represented as parameter types because it is not expected to have multiple types of `RuntimeEvent` or `WeightInfo` for a particular pallet within the same runtime. Instead, the associated types provide a way to associate specific types with the pallet's methods or associated functions, ensuring a clear and concise representation of the expected associations.

## Avoiding errors
Here're a couple of things to keep in mind in order to avoid errors when working with associated types and generics in substrate
- Ensure that all associated types required by the trait are implemented in your runtime
- Be mindful of the constraints placed on generic type parameters. Ensure that the constraints are appropriate and necessary for the desired functionality. Avoid overly restrictive constraints that may limit the usability of the code.
- Thoroughly test code that uses associated types and generic type parameters. Write unit tests that cover different scenarios and edge cases to ensure that the code behaves as expected and handles various inputs correctly.

## Summary
This guide discusses the usage of associated types and generic type parameters in Substrate. Associated types are used when you expect only one implementation of a trait for a given type, while generic type parameters are used when you anticipate having multiple implementations of a trait for different types.

The guide also provides examples of how associated types and generic type parameters are used in Substrate, such as defining traits with associated types and implementing them for specific types.

> Help us measure our progress and improve Substrate in Bits content by filling
out our living [feedback form](https://airtable.com/shr7CrrZ5zqlhWEUD).
Thank you!

