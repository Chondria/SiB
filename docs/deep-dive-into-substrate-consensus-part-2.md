---
tags:
  - substrate
keywords: [polkadot, substrate, consensus, finality, grandpa]
description: Deep dive into Substrate consensus - Part 2.
updated: 2023-11-16
author: cenwadike
duration: 2h
level: advanced
---

# Deep dive into Substrate consensus - part 2

We introduced why consensus is important to decentralized computing and
storage in [part-1](./deep-dive-into-substrate-consensus-part-1.md) of the
**deep dive into substrate consensus** series.

This guide continues where [part-1](./deep-dive-into-substrate-consensus-part-1.md)
ends. Here we will develop an understanding of blockchain finality, and have a
grasp of relevant Substrate components involved in block finality. We will
also look at why blockchain blocks must be finalized, and highlight common
approaches for block finality.

>Help us measure our progress and improve Substrate in Bits content by filling
out our living [feedback form](https://airtable.com/shr7CrrZ5zqlhWEUD).
Thank you!

## Review of Block processing

To provide context to the subsequent discussion, let us review how a block is
added to the blockchain. The steps below provide a summary of how a block is
created and added to the blockchain.

- Transactions are submitted to a node.

- Submitted transactions are validated and added to the transaction pool.

- Valid transactions are ordered based on Substrate's transaction priority
  system.

- Valid transactions are used to composed into a block, and a block header is
  created.

- The composed block gets executed by the runtime executor.

- Block is gossiped among peers and validated using the block header.

- An authorized node may add the executed block to the blockchain.

>You can find a detailed description with code examples of how blocks are
processed in a previous series [here](./from-transaction-to-block-part-2.md).

## Understanding Block Finality

You may have noticed that the steps above did not highlight how one can
ascertain the guarantee of the state of the blockchain at a given point in
time. We can say that there was no description of how we can know a block is
final and its content as well as its order in the blockchain will not change.

Block finality is the second face of a blockchain consensus mechanism.
(Block production being the first face of blockchain consensus). It solves the
problem of how can peers within a decentralized network know the correct chain
to build upon. By extension, it facilitates data consistency and enables
external data requests to produce the same result.

There are two widely used approaches to block finality; probabilistic vs
provable finality.

Probabilistic finality means that under some assumptions about the network and
participants, for example; if a few blocks build on a given block, one can
estimate the probability that it is final. Based on these assumptions, at some
point in the future, all nodes will agree on the truthfulness of one set of
data (ie. block becomes final). This eventual consensus on the finality of a
block may take a long time and will not be able to determine how long it will
take ahead of time.

Provable finality on the other hand depends on a set of specified rules and
processes to agree on the truthfulness of data on the blockchain. Under these
specified rules, peers within the network continue to agree on the truthfulness
of new and old blocks.

Finality gadgets such as GRANDPA (GHOST-based Recursive ANcestor Deriving
Prefix Agreement) or Ethereum's Casper FFG (the Friendly Finality Gadget) are
designed to give stronger and quicker guarantees on the finality of blocks -
specifically, that they can never be reverted after some process of Byzantine
agreements has taken place. The notion of irreversible consensus is known as
provable finality.

## GRANDPA

### GRANDPA design

GRANDPA (GHOST-based Recursive ANcestor Deriving Prefix Agreement) is the
finality gadget that is implemented by default for Substrate chains. Finality
is obtained by consecutive rounds of voting by validator nodes. Validators
execute the GRANDPA finality process in parallel to Block Production as an
independent service.

It is important to note that that GRANDPA reaches agreements on chains rather
than blocks which greatly speeds up the finalization process, even after
long-term network partitioning or other networking failures. This means that as
soon as more than 2/3 of validators attest to a chain containing a certain
block, all blocks leading up to that one are finalized at once.

The GRANDPA protocol specifies the following to successfully
participate in the block-finalization process:

- A **GRANDPA voter** is represented by a key pair generated using the
  ed25519 encryption protocol.
- **GRANDPA authorities** is a set of voters for a given block.
- The **GRANDPA authority set id** is an incremental counter that is updated
  under specific conditions.
- A **GRANDPA vote** is composed of a block hash and the block number.
- A **GRANDPA voter** can engage in a maximum of 2 sub-rounds of voting. The
  first sub-round is called the **pre-round** and the second sub-round is
  called **pre-commit**.
- **Voting** is done by broadcasting voting messages to the network.
- Validators inform their peers about the block finalized in a round by
  broadcasting a commit message.
- A **message** is a byte array containing the message to be signed.
- A **Vote signature** is the signature of a voter for a specific message in a
  round and is specific for each sub-round.
- A block must have a valid **justification**. The **justification** must
  contain up to one valid vote from each voter and must not contain more than
  two **equivocatory** votes from each voter.
- A **Voter** equivocates if they broadcast two or more valid votes to blocks
  during one voting sub-round.

To participate coherently in the voting process, a validator must initiate its
state and sync it with other active validators. In particular, considering that
voting happens in different distinct rounds where each round of voting is
assigned a unique sequential round number, a validator node needs to determine
and set its round counter equal to the voting round currently undergoing in the
network.

The process of joining a new voter set is different from the one of rejoining
the current voter set after a network disconnect.

For each round, an honest voter must participate in the voting process by
following **`Play-Grandpa-Round`**. It is important to note that we might not
always succeed in finalizing a block candidate in a round. In this case, another
round of voting is conducted before voting on the next voting round.

At the end of a voting round and block finalization, a **Justified Block Header**
for a block is appended to the blockchain. This *justified block header* contains
the following parts; the **`block header`**, **`justification`**, and
**`authority id`**.

To catch up to the latest chain after network disruption, a node (re)joins the
network, and requests the history of state transitions in the form of blocks,
which it is missing. When a **voter** node joins the network, it also needs to
gather the *justification* of the rounds it has missed. Through this process,
they can safely join the voting process of the current round, on which the
voting is taking place.

> check [here](https://spec.polkadot.network/sect-finality) for a full view of
GRANDPA specifications.

### GRANDPA implementation

From the section above, we can see that GRANDPA is fairly complex. Its
implemntation is contained mainly in three Substrate modules;
**`sp_consensus_grandpa`**, **`sc_consensus_grandpa`**, and **`pallet-grandpa`**.

The **`sp_consensus_grandpa`** and **`sc_consensus_grandpa`** module is part of
external client which facilitates running GRANDPA related consesus related
tasks. They contains all implementation GRANDPA related to peer-to-peer
communication, message and equivocation related tasks. The
**`sp_consensus_grandpa`** module also defines an interface used to interact
with **`pallet-grandpa`** from the external client.

The GRANDPA authority set is implemented like so:

```rust
pub struct AuthoritySet<H, N> {
    /// The current active authorities.
    pub(crate) current_authorities: AuthorityList,
    /// The current set id.
    pub(crate) set_id: u64,

    pub(crate) pending_standard_changes: ForkTree<H, N, PendingChange<H, N>>,

    pending_forced_changes: Vec<PendingChange<H, N>>,

    pub(crate) authority_set_changes: AuthoritySetChanges<N>,
}
```

The GRANDPA voter task which carry out voting mechanism is implemented like so:

```rust
/// Run a GRANDPA voter as a task. Provide configuration and a link to a
/// block import worker that has already been instantiated with `block_import`.
pub fn run_grandpa_voter<Block: BlockT, BE: 'static, C, N, S, SC, VR>(
    grandpa_params: GrandpaParams<Block, C, N, S, SC, VR>,
  ) -> sp_blockchain::Result<impl Future<Output = ()> + Send>
  where
    BE: Backend<Block> + 'static,
    N: NetworkT<Block> + Sync + 'static,
    S: SyncingT<Block> + Sync + 'static,
    SC: SelectChain<Block> + 'static,
    VR: VotingRule<Block, C> + Clone + 'static,
    NumberFor<Block>: BlockNumberOps,
    C: ClientForGrandpa<Block, BE> + 'static,
    C::Api: GrandpaApi<Block>,
  {
    let GrandpaParams {
      mut config,
      link,
      network,
      sync,
      voting_rule,
      prometheus_registry,
      shared_voter_state,
      telemetry,
      offchain_tx_pool_factory,
    } = grandpa_params;

    config.observer_enabled = false;

    let LinkHalf {
      client,
      select_chain,
      persistent_data,
      voter_commands_rx,
      justification_sender,
      justification_stream: _,
      telemetry: _,
    } = link;

    let network = NetworkBridge::new(
      network,
      sync,
      config.clone(),
      persistent_data.set_state.clone(),
      prometheus_registry.as_ref(),
      telemetry.clone(),
    );

    let conf = config.clone();
    let telemetry_task =
      if let Some(telemetry_on_connect) = telemetry.as_ref().map(|x| x.on_connect_stream()) {
        let authorities = persistent_data.authority_set.clone();
        let telemetry = telemetry.clone();
        let events = telemetry_on_connect.for_each(move |_| {
          let current_authorities = authorities.current_authorities();
          let set_id = authorities.set_id();
          let maybe_authority_id =
            local_authority_id(&current_authorities, conf.keystore.as_ref());

          let authorities =
            current_authorities.iter().map(|(id, _)| id.to_string()).collect::<Vec<_>>();

          let authorities = serde_json::to_string(&authorities).expect(
            "authorities is always at least an empty vector; \
            elements are always of type string",
          );

          telemetry!(
            telemetry;
            CONSENSUS_INFO;
            "afg.authority_set";
            "authority_id" => maybe_authority_id.map_or("".into(), |s| s.to_string()),
            "authority_set_id" => ?set_id,
            "authorities" => authorities,
          );

          future::ready(())
        });
        future::Either::Left(events)
      } else {
        future::Either::Right(future::pending())
      };

    let voter_work = VoterWork::new(
      client,
      config,
      network,
      select_chain,
      voting_rule,
      persistent_data,
      voter_commands_rx,
      prometheus_registry,
      shared_voter_state,
      justification_sender,
      telemetry,
      offchain_tx_pool_factory,
    );

    let voter_work = voter_work.map(|res| match res {
      Ok(()) => error!(
        target: LOG_TARGET,
        "GRANDPA voter future has concluded naturally, this should be unreachable."
      ),
      Err(e) => error!(target: LOG_TARGET, "GRANDPA voter error: {}", e),
    });

    // Make sure that `telemetry_task` doesn't accidentally finish and kill grandpa.
    let telemetry_task = telemetry_task.then(|_| future::pending::<()>());

    Ok(future::select(voter_work, telemetry_task).map(drop))
}
```

The **`pallet-grandpa`** manages the GRANDPA authority set on the runtime. It
routinely check for changes in the authority set change and enacts them in the
next voting round. The **`pallet-grandpa`** also implements equivocation offence
releted task.

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
- [GRANDPA](https://docs.rs/sc-consensus-grandpa/latest/sc_consensus_grandpa/)
- [GRANDPA spec](https://spec.polkadot.network/sect-finality)

>Help us measure our progress and improve Substrate in Bits content by filling
out our living [feedback form](https://airtable.com/shr7CrrZ5zqlhWEUD).
Thank you!
