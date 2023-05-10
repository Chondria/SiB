# Accessing Storage and Functions Across Custom Pallets

When developing blockchains with substrate, you’ll often need to use the functionalities provided by other pallets to save time in writing custom code. 
This guide will walk you through common problems faced when trying to access the functions of other pallets, as well as in-depth explanation of relevant concepts.


>To help us measure our progress and improve Substrate in Bits content, 
please fill our living [feedback form](https://airtable.com/appc45lFGS94WumrY/tblnuIR8lSd4TX7IR/viwqMQuAR6zSDn765?blocks=hide). 
It will only take 2 minutes of your time. Thank you!

## Reproducing errors

For the sake of this guide, we’ve created a custom blockchain and built a new pallet . The pallet allows users to make transfers only when certain conditions are met. Currently, the only functionality in this pallet is the `identity_transfer` method which ensures that only people with an on-chain identity can make transfers from their accounts.

### Environment and project set up.

To follow along with this tutorial, ensure that you have the rust toolchain installed

- Visit [substrate official documentation](https://docs.substrate.io/install/) page for the installation processes.
- Clone the project [repository](https://github.com/abdbee/Identity-based-transfer).

```rust
git clone https://github.com/abdbee/Identity-based-transfer.git
```

- Navigate into the project’s directory.

```rust
cd Identity-based-transfer
```

- Run the command below to compile the node.

```rust
cargo build --release
```

While attempting to compile the node above, you’ll encounter an error similar to the one below: 

![error.png](../Images/Screenshot%202023-05-09%20at%2008.38.05.png)

## Solving the error

The custom pallet for this project has one method (ie, the make-identity-transfer method) whose code you can find below:

```rust
 #[pallet::call]
	impl<T: Config> Pallet<T> {
		/// An example dispatchable that takes a singles value as a parameter, writes the value to
		/// storage and emits an event. This function must be dispatched by a signed extrinsic.
		#[pallet::call_index(0)]
		#[pallet::weight(10_000 + T::DbWeight::get().writes(1).ref_time())]
		pub fn make_identity_transfer(
            origin: OriginFor<T>,
            dest: AccountIdLookupOf<T>,
            #[pallet::compact] value: T::Balance,
        ) -> DispatchResultWithPostInfo {
			let sender = ensure_signed(origin)?;
            let dest = T::Lookup::lookup(dest)?;
			let lookup_dest = T::Lookup::unlookup(dest);
            ensure!(pallet_identity::Pallet::has_identity(&sender,1), Error::NotAuthorized);

            // Transfer the balance using the tightly coupled pallet-balances module
            pallet_balances::Pallet::transfer(OriginFor::from(Some(sender).into()), lookup_dest, value)?;
            Ok(().into())
		}
```

This function takes three parameters:

- **`origin`**: The origin of the call (the sender of the transaction).
- **`dest`**: The destination account ID in the lookup format.
- **`value`**: The balance to transfer, using the **`#[pallet::compact]`** attribute to optimize storage.

The function body does the following:

1. Ensures the **`origin`** is a signed sender (a user who has signed the transaction), and extracts the sender's account ID.
2. Converts the lookup format of the destination account ID (**`dest`**) into the actual account ID using **`T::Lookup::lookup(dest)?`**.
3. Reverts the actual account ID back to the lookup format using **`T::Lookup::unlookup(dest)`**.
4. Checks if the sender has the required identity with **`pallet_identity::Pallet::has_identity(&sender,1)`**. If not, it returns an error **`Error::NotAuthorized`**.
5. Transfers the specified **`value`** from the sender's account to the destination account using the tightly coupled **`pallet_balances::Pallet::transfer()`** function.
6. Returns **`Ok(().into())`** to indicate that the call was successful.

The error we’re getting above originates from 2 lines :

```rust
ensure!(pallet_identity::Pallet::has_identity(&sender,1), Error::NotAuthorized);
```

and

```rust
pallet_balances::Pallet::transfer(OriginFor::from(Some(sender).into()), lookup_dest, value)?;
```

By using the codes above, the compiler can not determine which implementation of the `Config` to use to use for the `identity` and `balances` pallets respectively in this specific runtime. This is because the required type parameter is not specified and the types are not hardcoded for the `Pallet` struct

In Substrate, the generic type parameter **`<T>`** is used to represent a specific implementation of the **`Config`** trait, which holds the configuration settings for the pallet in the runtime. Therefore, this needs to be specified when using components of these pallets.

We’ll go into more depth later, but for now, let’s solve the error.

- Replace
    
    ```rust
    ensure!(pallet_identity::Pallet::has_identity(&sender,1), Error::NotAuthorized);
    ```
    
    with 
    
    ```rust
    ensure!(pallet_identity::Pallet::<T>::has_identity(&sender,1), Error::NotAuthorized);
    ```
    
- Replace
    
    ```rust
    pallet_balances::Pallet::transfer(OriginFor::from(Some(sender).into()), lookup_dest, value)?;
    ```
    
    with 
    
    ```rust
    pallet_balances::Pallet::<T>::transfer(OriginFor::from(Some(sender).into()), lookup_dest, value)?;
    ```
    
- Re-compile the node
    
    ```rust
    cargo build --release
    ```
    
- 

## Going In-depth

### Assessing other pallets' features using trait bounds

In general, the features of other pallets can be assessed via two ways:

- Bounding the Config trait of the external pallet directly to your Pallet’s `Config` trait (Tight coupling)
- Bounding specific types in the external pallet to specific types in your Pallet’s `Config` trait (Loose coupling)

Essentially, were tight coupling provides the pallets access to all functionalities of the external pallet, loose coupling exposes only the required functionalities from the external pallet.

With tight coupling, a direct trait bound between the Config traits of the pallets involved is done. This exposes all the functionalities of the bound pallets, provided the pallets expose those functionalities through their Config trait.

Tight coupling was used in the example provided earlier, in which both the `identity` and `balances` pallets where bound to the `Config` trait of our custom pallet

```rust
#[pallet::config]
	pub trait Config: frame_system::Config + pallet_balances::Config + pallet_identity::Config {
		/// Because this pallet emits events, it depends on the runtime's definition of an event.
		type RuntimeEvent: From<Event<Self>> + IsType<<Self as frame_system::Config>::RuntimeEvent>;
	}
```

This essentially means that our custom pallet can access all the associated types, constants, and functions defined in the `Config` trait of the identity and balances pallets, as well as other types and functions that they expose via their `Config` trait. Tight coupling offers less modularity and flexibility because both modules must be included for one to be used, and any changes made in one pallet will often have an impact on the other

On the other hand, loose coupling allows you to selectively expose only the required functionality between pallets by having traits that has the required functionality and bounding these traits/types to the types in your Config trait that need to provide those functionalities. This means that changes to other parts of the pallets that aren’t exposed won’t have any impact on your runtime.

Loose coupling offers more modularity for pallets. To explain this, let’s use the `EnsureOrigin` trait in the democracy pallet.

[https://github.com/paritytech/substrate/blob/master/frame/democracy/src/lib.rs#L294-L352](https://github.com/paritytech/substrate/blob/master/frame/democracy/src/lib.rs#L294-L352)

```rust
/// Origin from which the next tabled referendum may be forced. This is a normal
		/// "super-majority-required" referendum.
		type ExternalOrigin: EnsureOrigin<Self::RuntimeOrigin>;

		/// Origin from which the next tabled referendum may be forced; this allows for the tabling
		/// of a majority-carries referendum.
		type ExternalMajorityOrigin: EnsureOrigin<Self::RuntimeOrigin>;
```

The `ExternalOrigin` type is bound to the `EnsurOrigin` trait. You’ll then have to specify the type to use in the runtime, and the type must implement the `EnsureOrigin` trait.

[https://github.com/paritytech/polkadot/blob/master/runtime/rococo/src/lib.rs#L383](https://github.com/paritytech/polkadot/blob/master/runtime/rococo/src/lib.rs#L383)

```rust
impl pallet_democracy::Config for Runtime {
/// A straight majority of the council can decide what their next motion is.
	type ExternalOrigin =
		pallet_collective::EnsureProportionAtLeast<AccountId, CouncilCollective, 1, 2>;
	/// A majority can have the next scheduled referendum be a straight majority-carries vote.
	type ExternalMajorityOrigin =
		pallet_collective::EnsureProportionAtLeast<AccountId, CouncilCollective, 1, 2>;
}
```

In the example above, the `EnsureProportionAtLeast` struct was used for the runtime implementation of the `ExternalOrigin` and `ExternalMajorityOrigin` types. This works because the `EnsureProportionAtLeast` struct implements the `EnsureOrigin` trait as shown below:

[Source](https://github.com/paritytech/substrate/blob/master/frame/collective/src/lib.rs#LL1207C1-L1224C3)

```rust
pub struct EnsureProportionAtLeast<AccountId, I: 'static, const N: u32, const D: u32>(
	PhantomData<(AccountId, I)>,
);
impl<
		O: Into<Result<RawOrigin<AccountId, I>, O>> + From<RawOrigin<AccountId, I>>,
		AccountId,
		I,
		const N: u32,
		const D: u32,
	> EnsureOrigin<O> for EnsureProportionAtLeast<AccountId, I, N, D>
{
	type Success = ();
	fn try_origin(o: O) -> Result<Self::Success, O> {
		o.into().and_then(|o| match o {
			RawOrigin::Members(n, m) if n * D >= N * m => Ok(()),
			r => Err(O::from(r)),
		})
	}
```

If in the future you’ll like to use a different pallet for the runtime implementation of `ExternalOrigin` and `ExternalMajorityOrigin` , all you’ll have to do is to declare a new struct in the pallet, implement the `EnsurOrigin` trait for the new struct and assign it to `ExternalOrigin` and `ExternalMajorityOrigin` when implementing the pallet in your runtime.  

### A Deeper look at the error encountered

Let’s now take a deeper look at the error encountered when trying to use a function from another pallet without specifying the generic type parameter `<T>`

All substrate pallets have a `Pallet` struct which acts as a container for all the items related to the pallet. This struct has at least one generic type parameter `<T>`.

```rust
#[pallet::pallet]
	pub struct Pallet<T>(_);
```

The type for all implementations for this struct must be bound to the pallet’s `config` trait.

```rust
impl<T: Config> Pallet<T> {
	pub fn some_function() {
        // code here
    }
}
```

This means that a pallet’s `Config` trait must be implemented in any runtime that wants to use the functions of the pallet’s struct. This implementation must contain all the necessary configurations for the types and constants exposed by the pallet in the `Config` trait.

Since the pallet uses a generic type parameter, it becomes mandatory to add this parameter when calling a function from an external pallet

```rust
let some_return_vaue = pallet_name::Pallet::<T>::some_function();
```

The `<T>` refers to the runtime type. Not adding  `<T>` will lead to an error because the compiler wouldn’t know the specific type to use for the generic type parameter  `<T>` in your pallet’s struct.

But by adding `<T>`, you’re telling the compiler to use the specific configuration provided by your runtime for the pallet. It can then use this configuration to infer the types and constants to use for the function you called.

Note that it would also have been possible to hardcode the types in the pallet rather than use generic type parameters. But this wouldn’t be good design for some reasons:

- Without generic type parameters, it would be hard to configure the pallet according to specific runtime requirements
- The pallets would be harder to re-use because the types are hard-coded and cannot be easily adapted to work with other types.
- There’s lesser flexibility when configuring different instances of the same pallet.

## Summary

In this guide, you:

- Encountered a compilation error in the template due to not specifying the generic type parameter **`<T>`** when using functions from external pallets.
- To fix the error, you replaced the two problematic lines of code with the appropriate lines that include the generic type parameter **`<T>`**.

Also, you learned that:

- All Substrate pallets have a Pallet struct with at least one generic type parameter **`<T>`**.
- The implementations for the Pallet struct must be bound to the pallet's **`Config`** trait.
- A pallet's Config trait must be implemented in any runtime that wants to use the functions of the pallet's struct.
- When calling a function from an external pallet, the generic type parameter **`<T>`** must be specified.
- Not specifying **`<T>`** will lead to a compilation error, as the compiler wouldn't know the specific type to use for the generic type parameter **`<T>`** in the pallet's struct.
- By adding **`<T>`**, the compiler can use the specific configuration provided by the runtime for the pallet to infer the types and constants to use for the called function.
- Hardcoding types in the pallet instead of using generic type parameters is not a good design because it:
    - Makes it difficult to configure the pallet according to specific runtime requirements.
    - Reduces reusability, as the hardcoded types cannot be easily adapted to work with other types.
    - Offers less flexibility when configuring different instances of the same pallet.
- Tight coupling and loose coupling are two approaches to access features of other pallets in Substrate.
- Tight coupling involves binding the Config trait of the external pallet directly to your pallet's Config trait, providing access to all functionalities of the bound pallets.
- Loose coupling involves binding specific types or traits in the external pallet to specific types in your pallet's Config trait, selectively exposing only the required functionalities.
- Tight coupling offers less modularity and flexibility, while loose coupling provides better modularity and adaptability.
- The democracy pallet example demonstrated the loose coupling approach using the **`EnsureOrigin`** trait.
- Loose coupling allows for easier future modifications, as changing the implementation only requires implementing a new struct and assigning it when implementing the pallet in runtime.

To learn more about the concepts discussed in this guide, here are some resources that we recommend:

- [Pallet coupling](https://docs.substrate.io/build/pallet-coupling/)
- [Custom pallets](https://docs.substrate.io/build/custom-pallets/)
- [Tight coupling](https://docs.substrate.io/reference/how-to-guides/pallet-design/use-tight-coupling/)

We’re inviting you to fill our living [feedback form](https://airtable.com/shr7CrrZ5zqlhWEUD) to help us measure our progress and improve Substrate in Bits content. It will only take 2 minutes of your time. Thank you!
