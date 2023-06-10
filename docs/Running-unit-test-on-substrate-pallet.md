---
tags:
  - substrate
keywords: [polkadot, substrate, pallets, unit test, mock runtime, rust]
description: Running unit tests on substrate pallet.
updated: 2023-06-11
author: cenwadike
duration: 3h
level: intermediate
---

# Running unit tests on substrate pallet
Testing the functionality of any software is an essential component of a software development lifecycle. Unit testing in substrate allows you to confirm that the methods exposed by a pallet are logically correct. Testing also allows you to ascertain that data related to a pallet is handled correctly when interacting with the pallet.

Substrate provides a comprehensive set of APIs that allow you to set up a test environment. This test environment can mock substrate runtime and simulate transaction execution for extrinsics and queries of your runtime.  

In this guide, we will walk through a common problem related to mocking a runtime and testing a substrate pallet. We will also have an in-depth look at some important APIs that substrate exposes for testing and how to leverage them for complex testing scenarios.

>Help us measure our progress and improve Substrate in Bits content by filling out our living [feedback form](https://airtable.com/appc45lFGS94WumrY/tblnuIR8lSd4TX7IR/viwqMQuAR6zSDn765?blocks=hide). Thank you!


## Reproducing errors

### Environment and project setup

To follow along with this tutorial, ensure that you have the Rust toolchain installed.

- Visit the [substrate official documentation](https://docs.substrate.io/install/) page for the installation processes.

- Clone the project [repository](https://github.com/cenwadike/Running-unit-test-on-substrate-pallet).

```rust
git clone https://github.com/cenwadike/Running-unit-test-on-substrate-pallet
```

- Navigate into the project’s directory.

```
cd Running-unit-test-on-substrate-pallet
```

Checkout to faulty test.

```
git checkout 1a7a90c
```

- Run the command below to build the pallet.

```
cargo build --release
```

- Run the command below to test the pallet.

```
cargo test 
```

While attempting to run the test, you’ll encounter an error like the one below:

```bash
error[E0046]: not all trait items implemented, missing: `RuntimeEvent`
  --> src/mock.rs:52:1
   |
52 | impl pallet_archiver::Config for Test {}
   | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ missing `RuntimeEvent` in implementation
   |
  ::: src/lib.rs:46:9
   |
46 |         type RuntimeEvent: From<Event<Self>> + IsType<<Self as frame_system::Config>::RuntimeEvent>;
   |         ------------------------------------------------------------------------------------------- `RuntimeEvent` from trait
```

This compiler error tells us that **`RuntimeEvent`** is not implemented on the mock runtime. The error further points us to the Pallet **`Config`** trait in `lib.rs` where **`RuntimeEvent`** is defined.

We may recall that substrate exposes a rich set of APIs that allows us to mock a runtime without having to scaffold a full-blown runtime ourselves. However, to couple our pallet to this mock runtime, we must implement the **`Config`** trait of our pallet on the mock runtime.

A careful inspection of `mock.rs` reveals that a mock runtime was constructed using the `FRAME` **`construct_runtime`** macro taking **`Test`** enum as an argument for the mock runtime. This **`Test`** must contain implementations for each of the pallet configuration traits that are used in the mock runtime.

The `Test` enum in `mock.rs` contains the trait definition for the `frame_system` pallet and our custom `archiver_pallet` like so:

```rust 
// Configure a mock runtime to test the pallet.
frame_support::construct_runtime!(
    pub enum Test where
        Block = Block,
        NodeBlock = Block,
        UncheckedExtrinsic = UncheckedExtrinsic,
    {
        System: frame_system,
        ArchiverPallet: pallet_archiver,
    }
);
```
Because `Test` only needs to define the pallet, each pallet's configuration trait must be implemented (separately).
As you may have observed in `mock.rs`, each pallet's `Config` trait was implemented for `Test` and all relevant `Config` types were defined for each.

Our error resulted from the missing implementation of the `RuntimeEvent` trait type in the `archiver_pallet` `Config` implementation.

## Solving the error
The solution to the compiler error is to implement **`RuntimeEvent`** trait type like so:

```rust
impl pallet_archiver::Config for Test {
    type RuntimeEvent = RuntimeEvent;     // <-- add this line
}
```

- Re-run the test

```rust
cargo test
```

## Going in-depth
In the previous section, we focused on how we can use substrate APIs to construct a mock runtime for tests. What we did not comment  on, is how to mimic transactions and storage queries.

Substrate exposes an I/O interface that enables you to run several tests independently of each other through **`sp_io::TestExternalities`**.  `sp_io::TestExternalities` is an alias of `sp_state_machine::TestExternalities`. 

`sp_state_machine::TestExternalities` can be viewed as a class that implements a wide array of methods that allows you to interact with a mock runtime. 

`sp_state_machine::TestExternalities` is implemented as a `struct` with an `impl` block. Its `impl` contains several helpful methods that can be employed in different test scenarios. A particular commonly used method is **`execute_with`**. 

```rust
impl<H> TestExternalities<H>
where
	H: Hasher + 'static,
	H::Out: Ord + 'static + codec::Codec,
{
   // ----- *snip* ------

   pub fn execute_with<R>(&mut self, execute: impl FnOnce() -> R) -> R {
		let mut ext = self.ext();
		sp_externalities::set_and_run_with_externalities(&mut ext, execute)
	}

   // ----- *snip* ------
}
```

`execute_with` exposes the `sp_state_machine::TestExternalities` constructed from our `Test` enum to a test case and simulates the execution of a substrate extrinsic.

From this, we can construct a test for `archiver_pallet` like so:

```rust
#[test]
fn archive_book_works() {
    new_test_env().execute_with(|| {
      // ----- *snip* ------

      assert_ok!(ArchiverPallet::archive_book(
         RuntimeOrigin::signed(1),
         title.clone(),
         author.clone(),
         url.clone(),
      ));

      // ----- *snip* ------
    });
}
```

We can observe that `sp_state_machine::TestExternalities` allows us to also query the storage of the mock runtime like so:

```rust
#[test]
fn archive_book_works() {
    new_test_env().execute_with(|| {
      // ----- *snip* ------

      let stored_book_summary = ArchiverPallet::book_summary(hash).unwrap();
      assert_eq!(stored_book_summary.url, url);
    });
}
```

It is important to know that every step of testing in substrate can be customized for your specific use case. This includes adding external pallets as dependencies on the mock runtime. 

A look at the `construct_runtime` documentation gives a clue about how we can include external pallets into our mock runtime. We can do this like so:

```rust
// Configure a mock runtime to test the pallet.
frame_support::construct_runtime!(
   pub enum Test where
      Block = Block,
      NodeBlock = Block,
      UncheckedExtrinsic = UncheckedExtrinsic,
   {
      System: frame_system,
      ArchiverPallet: pallet_archiver,
      AnotherPallet: path::to::pallet_another,   // <-- another pallet trait type
      System: frame_system::{Pallet, Call, Event<T>, Config<T>} = 0,
        Test: path::to::test::{Pallet, Call} = 1,
   }
);
```

```rust
// Implement another pallet Config trait
impl pallet_another::Config for Test {
   type ConfigType = ConcreteConfigType;
}
```
>You should also ensure that all external pallets are added as dependencies in `Cargo.toml` file and imported into mock.rs.

You can also further specify what parts of a pallet you need in your mock runtime like so:
```rust
// Configure a mock runtime to test the pallet.
frame_support::construct_runtime!(
   pub enum Test where
      Block = Block,
      NodeBlock = Block,
      UncheckedExtrinsic = UncheckedExtrinsic,
   {
      System: frame_system::{Pallet, Call, Event<T>, Config<T>},
      ArchiverPallet: pallet_archiver,
      AnotherPallet: path::to::pallet_another::{Pallet, Call},   
   }
);
```

## Summary
In this guide, we broke down a common problem faced when mocking a runtime for testing substrate pallets. We developed an understanding of:
- how to create a mock runtime for different test scenarios.
- how to include custom and external pallets in a mock runtime.
- how to mimic substrate extrinsic on a mock runtime.
- how to query the storage of a mock runtime.

We also had a look at important substrate APIs including:
- frame_system `construct_runtime` macro for building a runtime from provided pallets.
- `sp_io::TestExternalities` for interacting with a mock runtime through a test case.

This article was focused on testing the functionalities of pallets, we did **not** learn how to test a full node. In a future article, we will look at how to implement test scenarios on full node, so keep an eye out for this.

To learn more about testing in substrate, check out these resources:
- [Unit test](https://docs.substrate.io/test/unit-testing/#test-pallet-log-in-a-mock-runtime)
- [TestExternalities](https://docs.rs/sp-state-machine/0.28.0/sp_state_machine/struct.TestExternalities.html#impl-From%3CT%3E-for-TestExternalities%3CH%3E)
- [construct_runtime](https://docs.rs/frame-support/21.0.0/frame_support/macro.construct_runtime.html)


> We’re inviting you to fill out our living [feedback form](https://airtable.com/shr7CrrZ5zqlhWEUD) to help us measure our progress and improve Substrate in Bits content. It will only take 2 minutes of your time. Thank you!










