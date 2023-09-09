---
tags:
  - substrate
keywords: [polkadot, substrate, pallets, currency, fungible tokens, balance, account, rust]
description: Working with assets on substrate.
updated: 2023-06-27
author: cenwadike
duration: 3h
level: intermediate
---

# Working with assets on substrate

It is common practice to store units of value on the blockchain as fungible
assets. Fungible assets facilitate the implementation of a sound economic game
model for a blockchain and provide a medium for exchanging value among
different participants within a blockchain economy.

This guide takes an error-based approach to walk you through essential
components when handling fungible assets in a substrate pallet. We will also
highlight important substrate APIs and FRAME pallets to consider when working
with fungible assets.

>Help us measure our progress and improve Substrate in Bits content by filling
out our living [feedback form](https://airtable.com/shr7CrrZ5zqlhWEUD).
Thank you!

## Reproducing errors

### Environment and project setup

To follow along with this tutorial, ensure that you have the Rust toolchain
installed.

- Visit the [substrate official documentation](https://docs.substrate.io/install/)
page for the installation processes.

- Clone the project [repository](https://github.com/cenwadike/working-with-assets-on-substrate).

```rust
git clone https://github.com/cenwadike/working-with-assets-on-substrate
```

- Navigate into the project’s directory.

```bash
cd working-with-assets-on-substrate
```

Checkout to faulty implementation.

```bash
git checkout 858eb27
```

- Run the command below to build the pallet.

```bash
cargo build --release
```

Running the build produces an error like the one below:

```bash
error[E0220]: associated type `Currency` not found for `T`
   --> src/lib.rs:203:24
    |
203 |                     T::Currency::reserve(&signer, deposit.into())
    |                        ^^^^^^^^ there is a similarly named associated type `Currency` in the trait `VestingSchedule`
```

This compiler error tells us that **`Currency`** type is not implemented for
the pallet config trait **`T`**. The error further suggests that there is a
type with a similar name **`Currency`** in the trait **`VestingSchedule`** that
we could use in the pallet.

For context, the error originates from `pay_royalty` extrinsic that 'accepts' a
royalty payer asset and reserves such asset for the archiver. The archiver can
withdraw this asset into their account.

## Solving the error

Similar to the common fiat lingo, `Currency` is an abstraction over fungible
asset systems. It defines essential methods used in handling fungible assets
including `transfer`, `withdraw` and `burn`.

Currency trait is implemented like so:

```rust
/// Abstraction over a fungible assets system.
pub trait Currency<AccountId> {
 /// The balance of an account.
 type Balance: Balance + MaybeSerializeDeserialize + Debug + MaxEncodedLen + FixedPointOperand;

 /// The opaque token type for an imbalance. This is returned by unbalanced operations
 /// and must be dealt with. It may be dropped but cannot be cloned.
 type PositiveImbalance: Imbalance<Self::Balance, Opposite = Self::NegativeImbalance>;

 /// The opaque token type for an imbalance. This is returned by unbalanced operations
 /// and must be dealt with. It may be dropped but cannot be cloned.
 type NegativeImbalance: Imbalance<Self::Balance, Opposite = Self::PositiveImbalance>;

    //-----------------snip---------------------//

 /// The combined balance of `who`.
 fn total_balance(who: &AccountId) -> Self::Balance;

 fn burn(amount: Self::Balance) -> Self::PositiveImbalance;

 fn slash(who: &AccountId, value: Self::Balance) -> (Self::NegativeImbalance, Self::Balance);

 fn deposit_into_existing(
  who: &AccountId,
  value: Self::Balance,
 ) -> Result<Self::PositiveImbalance, DispatchError>;

 fn withdraw(
  who: &AccountId,
  value: Self::Balance,
  reasons: WithdrawReasons,
  liveness: ExistenceRequirement,
 ) -> Result<Self::NegativeImbalance, DispatchError>;

    //-----------------snip---------------------//
}
```

`Currency` along with `ReservableCurrency` (and other traits) are essential for
a comprehensive management of fungible assets as demonstrated in
[pallet-balance](https://docs.rs/pallet-balances/latest/pallet_balances/index.html).

Fixing this error above requires that we use `Currency` and `ReservableCurrency`
in our pallet `Config` like so:

```rust
#[frame_support::pallet]
pub mod pallet {
    use frame_support::traits::{Currency, ReservableCurrency};

    //--------------------snip------------------------//

    #[pallet::config]
    pub trait Config: frame_system::Config {
        /// Because this pallet emits events, it depends on the runtime's definition of an event.
        type RuntimeEvent: From<Event<Self>> + IsType<<Self as frame_system::Config>::RuntimeEvent>;
        /// Trait for handling fungible tokens
        type Currency: Currency<Self::AccountId> + ReservableCurrency<Self::AccountId>;
        /// Pallet ID.
        #[pallet::constant]
        type PalletId: Get<PalletId>;
    }

    //--------------------snip------------------------//

}
```

- Re-run the build

```rust
cargo build
```

It is important to note that `FRAME pallet-balance` further abstracts common
functionalities when handling fungible assets including transferring balance
between accounts, setting locks on balance, and reserving assets, as well as
slashing an account balance, making it easier for substrate pallet developers
to implement custom state transition functions involving fungible assets.

### Common pitfall

A common challenge you may face when implementing state transition functions
involving fungible assets is using a root account. A root account in this
context is an account that has full access to assets owned by a pallet.

An approach to implementing a root account is by converting your `PalletId` to
an `AccountId`. This account id can be associated with asset balances and can
make transfers with `admin` privileges.

You can implement a root account like so:

```rust
    // generate pallet account id
    pub fn pallet_account_id() -> T::AccountId {
        T::AccountId = T::PalletId::get().into_account_truncating()       
    }
```

Note that conversion from `PalletId` consumes computational resources and
should be accounted for during your runtime benchmarking.

If you need to use the pallet account id multiple times, you may want to
consider storing the pallet account id after a single computation as a
single StorageValue like so:

```rust
    // Pallet derived account id
    #[pallet::storage]
    pub(super) type PalletAccountId<T: Config> = StorageValue<_, T::AccountId, OptionQuery>;

    // generate pallet account id
    pub fn pallet_account_id() -> T::AccountId {
        let pallet_account_id: T::AccountId = T::PalletId::get().into_account_truncating();
        PalletAccountId::<T>::put(pallet_account_id.clone());
        pallet_account_id
    }
```

You can initialize the pallet account id ideally in `GenesisConfig`.

## Summary

In this guide, we took an error-based approach to explore essential components
commonly used in handling fungible assets. We also developed an understanding of:

- Currency trait, which contains basic functionalities including asset transfers
- FRAME pallet-balances, that abstract a comprehensive set of fungible asset
functionalities

Additionally, we looked at how to use PalletId as a seed to generate a root
account id for a pallet.

To learn more about testing in substrate, check out these resources:

- [Currency](https://docs.rs/frame-support/21.0.0/frame_support/traits/tokens/currency/trait.Currency.html)
- [FRAME pallet-balances](https://docs.rs/pallet-balances/latest/pallet_balances/index.html)

> We’re inviting you to fill out our living
[feedback form](https://airtable.com/shr7CrrZ5zqlhWEUD) to help us measure our
progress and improve Substrate in Bits content.
It will only take 2 minutes of your time. Thank you!
