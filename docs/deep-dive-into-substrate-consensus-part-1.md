---
tags:
  - substrate
keywords: [polkadot, substrate, consensus, aura, babe]
description: Deep dive into Substrate consensus - Part 1.
updated: 2023-10-31
author: cenwadike
duration: 2h
level: advanced
---

# Deep dive into Substrate consensus - part 1

Blockchains essentially record data in multiple _computers_ and serve this
data in a manner that mimics a single _computer_. As a result, it needs an
approach to ensure that recorded data at any given point in time will continue
to exist without any modification.

A consensus mechanism describes who can permanently record data, a provides a
guarantee that the data will continue to exist in its original form.

In this guide, we will review important blockchain consensus concepts and
navigate how a couple of consensus components are implemented in Substrate. We
will highlight some code and explain how different components fit together.

>Help us measure our progress and improve Substrate in Bits content by filling
out our living [feedback form](https://airtable.com/shr7CrrZ5zqlhWEUD).
Thank you!

## Overview of Blockchain Consensus

Detailed articles about consensus mechanisms leverage in Substrate have been
provided [here](https://medium.com/polkadot-network/polkadot-consensus-part-1-introduction-3e3cd6237243)
and [here](https://wiki.polkadot.network/docs/learn-consensus#why-do-we-need-consensus).
In this section, we will highlight some important concepts relevant to general
concept consensus and briefly mention out-of-box consensus mechanisms made
available in Substrate.

Consensus mechanisms describe how the state transition changes reach some form
of guarantee. This guarantee provides the base of trust in the correctness of
data on a blockchain. Because this guarantee of correctness involves some form
of corporative participation between different nodes in the blockchain,
consensus mechanisms are tightly connected to how active, or alive a network is.
A blockchain is said to be lively if it is currently executing transactions and
adding changes from processed transactions to the blockchain.

We may be able to see that consensus has a couple of mandates it must satisfy
to ensure data security and a lively blockchain. Because the majority of data
stored on a blockchain is related to the transfer of value, data security is
equivalent to value security. A consensus bug can be fatal and in the best of
cases can halt transaction processing and can be difficult to recover from
without a full network reboot.

In a nutshell, blockchain defines the following:

1. What a valid chain is.
2. Who can add a block to a valid chain.
3. How to solve chain fork and recovery after partial or total network outage.

Substrate consensus implementation strategy decouples (2) form (1) and (3).
`Who can add a new block` is a question of who has the authority to "pen down"
a block into the valid chain. Substrate offers two production-ready consensus
authority modules **BABE** and **AURA** which can be used differently to grant
authority to a node that can add a new block.

What is considered a valid chain is handled in Substrate by **GRANDPA**.
**GRANDPA** is a finality gadget used by Substrate to define
`what a valid chain is` and what chain a node should build it chain from in
case of a network outage.

> _More details on AURA, BABE, and GRANDPA are discussed in subsequent sections_
[here](#block-authoring-protocols) and [here](#substrate-finality-gadget).

> _Dive into how transactions are executed in our guide [here](./from-transaction-to-block-part-1.md)_

## Block Authoring Protocols

Transactions are grouped together and collectively stored as blocks. Every node
can create blocks and execute transactions, however, only full nodes
participate in consensus. At the end of a specified duration of time, a full
node under specific conditions may be authorized to add the next block. The
added block is recognized as the latest block of the valid chain. The node that
adds a block is said to be the author of the block.

Usually, when a full node joins the blockchain network, its identity may be
added to a list of identities that can be selected to author a block. The
specifics of who gets added to this list and how members of the list are
selected are defined by the authoring protocols.

Substrate modular structure allows developers to use customized authoring
protocols implemented in part as FRAME pallets. The out-the-box block authoring
mechanisms provided by Substrate include:

1. Authority-based round-robin scheduling
2. Blind assignment of blockchain extension (BABE) slot-based scheduling.
3. Proof of work computation-based scheduling.

> _The specific duration of time before the next block authoring may be
referred to as **Block Time** and **Slot**, and by default is 6sec in Substrate_.

> _It is important that although the default block time is 6 sec, block
execution and authoring usually take less than 6sec. If you intended this
value, you can check for **MILLISECS_PER_BLOCK** in `runtime/src/lib.rs`._

> _Ensure you thoroughly your runtime if you modify this value on the minimum
hardware for your node._

> Check out how to benchmark runtime pallets [here](./Benchmarking-substrate-pallet.md)

### Aura Implementation

Aura (Authority-round) works by having a list of authorities A who are expected
to roughly agree on the current time. Time is divided up into discrete slots of
t seconds each. For each slot, the author of that slot is a node in the
authority list. The author is allowed to issue one block but not more during that
slot, and it will be built upon the longest valid chain that has been seen.

Aura is implemented across two crates; `sc_consensus_aura` and
`pallet_aura`. `sc_consensus_aura` crate contains the outer client
specifications of Aura while `pallet_aura` is a runtime extension of Aura.

[`sc_consensus_aura`](https://docs.rs/sc-consensus-aura/latest/sc_consensus_aura/)
primarily handles how blocks are handled. This includes starting an Aura worker,
importing Blocks from the Block queue, block verification, and view calls to
runtime for the authority list. During the slot of a node, it is this part of
the Substrate client that carries out the actual block authoring and propagation.

[`pallet_aura`](https://crates.parity.io/pallet_aura/index.html) handles how
authorities are selected for a slot. This includes authority rotation and
authority validation. This pallet uses the on_initialize hook to ensure a
current authority list like so:

```rust
    // -------------snip----------

#[pallet::pallet]
pub struct Pallet<T>(sp_std::marker::PhantomData<T>);

#[pallet::hooks]
impl<T: Config> Hooks<BlockNumberFor<T>> for Pallet<T> {
    fn on_initialize(_: T::BlockNumber) -> Weight {
        if let Some(new_slot) = Self::current_slot_from_digests() {
        let current_slot = CurrentSlot::<T>::get();

        assert!(current_slot < new_slot, "Slot must increase");
        CurrentSlot::<T>::put(new_slot);

        if let Some(n_authorities) = <Authorities<T>>::decode_len() {
                let authority_index = *new_slot % n_authorities as u64;
                if T::DisabledValidators::is_disabled(authority_index as u32) {
                panic!(
                "Validator with index {:?} is disabled and should not be attempting to author blocks.",
                authority_index,
                );
                }
            }

            // TODO [#3398] Generate offence report for all authorities that skipped their
            // slots.

            T::DbWeight::get().reads_writes(2, 1)
        } else {
            T::DbWeight::get().reads(1)
        }
    }
}

    // -------------snip----------
```

> Aura is the default block authoring mechanism of Substrate node template.

> You can check out how exactly blocks are propagated within a blockchain
network [here](./from-transaction-to-block-part-2.md)
> We also demonstrated how you can use your custom hooks
[here](./working-with-hooks.md)

### BABE Implementation

BABE (Blind Assignment for Blockchain Extension) is a slot-based block
production mechanism that uses a verifiable random function (VRF) to
perform slot allocation. On every slot, all the authorities generate a new
random number with the VRF, and if it is lower than a given threshold (which is
proportional to their weight/stake) they have a right to produce a block. The
proof of the VRF function execution will be used by other peers to validate the
legitimacy of the slot claim.

Similar to [Aura](#aura-implementation), BABE is implemented across two crates; the
`sc_consensus_babe` crate contains the outer client specifications of BABE
while `pallet_babe` is a runtime extension of BABE.

[`sc_consensus_babe`](https://docs.rs/sc-consensus-babe/latest/sc_consensus_babe/#)
handles block-specific operations including block propagation, block
verification, and adding new blocks to the valid chains of blocks. It also
handles slot-specific operations including random value requests from
`pallet_babe`, slot claiming, and epoch changes storage. It is also the
`sc_consensus_babe` workers that orchestrate BABE block authoring operations
including BABE consensus key handling.

[`pallet_babe`](https://crates.parity.io/pallet_babe/index.html) Collects
on-chain randomness from VRF and handles epoch transitions by the outer client.
It initializes and maintains the current list of authorities.

The main actions of `pallet_babe` are triggered at the beginning and end of a
block execution like so:

```rust
    // -------------snip-------------

#[pallet::hooks]
impl<T: Config> Hooks<BlockNumberFor<T>> for Pallet<T> {
    /// Initialization
    fn on_initialize(now: BlockNumberFor<T>) -> Weight {
        Self::do_initialize(now);
        0
    }

    /// Block finalization
    fn on_finalize(_n: BlockNumberFor<T>) {
        // at the end of the block, we can safely include the new VRF output
        // from this block into the under-construction randomness. If we've determined
        // that this block was the first in a new epoch, the changeover logic has
        // already occurred at this point, so the under-construction randomness
        // will only contain outputs from the right epoch.
        if let Some(Some(randomness)) = Initialized::<T>::take() {
            Self::deposit_randomness(&randomness);
        }

        // remove temporary "environment" entry from storage
        Lateness::<T>::kill();
    }
}

    // -------------snip-------------

fn do_initialize(now: T::BlockNumber) {
    // since do_initialize can be called twice (if session module is present)
    // => let's ensure that we only modify the storage once per block
    let initialized = Self::initialized().is_some();
    if initialized {
        return
    }

    let maybe_pre_digest: Option<PreDigest> =
        <frame_system::Pallet<T>>::digest()
            .logs
            .iter()
            .filter_map(|s| s.as_pre_runtime())
            .filter_map(|(id, mut data)| {
                if id == BABE_ENGINE_ID {
                    PreDigest::decode(&mut data).ok()
                } else {
                    None
                }
            })
            .next();

    let is_primary = matches!(maybe_pre_digest, Some(PreDigest::Primary(..)));

    let maybe_randomness: MaybeRandomness = maybe_pre_digest.and_then(|digest| {
        // on the first non-zero block (i.e. block #1)
        // this is where the first epoch (epoch #0) actually starts.
        // we need to adjust internal storage accordingly.
        if *GenesisSlot::<T>::get() == 0 {
            GenesisSlot::<T>::put(digest.slot());
            debug_assert_ne!(*GenesisSlot::<T>::get(), 0);

            // deposit a log because this is the first block in epoch #0
            // we use the same values as genesis because we haven't collected any
            // randomness yet.
            let next = NextEpochDescriptor {
                authorities: Self::authorities(),
                randomness: Self::randomness(),
            };

            Self::deposit_consensus(ConsensusLog::NextEpochData(next))
        }

        // the slot number of the current block being initialized
        let current_slot = digest.slot();

        // how many slots were skipped between current and last block
        let lateness = current_slot.saturating_sub(CurrentSlot::<T>::get() + 1);
        let lateness = T::BlockNumber::from(*lateness as u32);

        Lateness::<T>::put(lateness);
        CurrentSlot::<T>::put(current_slot);

        let authority_index = digest.authority_index();

        if T::DisabledValidators::is_disabled(authority_index) {
            panic!(
                "Validator with index {:?} is disabled and should not be attempting to author blocks.",
                authority_index,
            );
        }

        // Extract out the VRF output if we have it
        digest.vrf_output().and_then(|vrf_output| {
            // Reconstruct the bytes of VRFInOut using the authority id.
            Authorities::<T>::get()
                .get(authority_index as usize)
                .and_then(|author| schnorrkel::PublicKey::from_bytes(author.0.as_slice()).ok())
                .and_then(|pubkey| {
                    let transcript = sp_consensus_babe::make_transcript(
                        &Self::randomness(),
                        current_slot,
                        EpochIndex::<T>::get(),
                    );

                    vrf_output.0.attach_input_hash(&pubkey, transcript).ok()
                })
                .map(|inout| inout.make_bytes(&sp_consensus_babe::BABE_VRF_INOUT_CONTEXT))
        })
    });

    // For primary VRF output we place it in the `Initialized` storage
    // item and it'll be put onto the under-construction randomness later,
    // once we've decided which epoch this block is in.
    Initialized::<T>::put(if is_primary { maybe_randomness } else { None });

    // Place either the primary or secondary VRF output into the
    // `AuthorVrfRandomness` storage item.
    AuthorVrfRandomness::<T>::put(maybe_randomness);

    // enact epoch change, if necessary.
    T::EpochChangeTrigger::trigger::<T>(now)
}

    // -------------snip-------------
```

## Summary

We were able to get a firm grasp of the concept of blockchain consensus
especially how the block authoring works within the framework of Substrate.
We appreciated how Substrate implemented out-of-the-box block authoring
mechanisms and gained insight into relevant Substrate consensus modules.

We developed an understanding of the following:

- Why blockchains need consensus.
- Functional components of a  consensus mechanism.
- Aura implementation.
- BABE implementation.

To learn more about substrate consensus, check out these resources:

- [Consensus Wiki](https://medium.com/polkadot-network/polkadot-consensus-part-1-introduction-3e3cd6237243)
- [W3F Consensus Series](https://wiki.polkadot.network/docs/learn-consensus#why-do-we-need-consensus)
- [Substrate Consensus](https://docs.substrate.io/learn/consensus/#consensus-in-two-phases)
- [Authority](https://docs.substrate.io/reference/glossary/#authority)
- [Aura](https://docs.rs/sc-consensus-aura/latest/sc_consensus_aura/)
- [BABE](https://docs.rs/sc-consensus-babe/latest/sc_consensus_babe/)

>Help us measure our progress and improve Substrate in Bits content by filling
out our living [feedback form](https://airtable.com/shr7CrrZ5zqlhWEUD).
Thank you!
