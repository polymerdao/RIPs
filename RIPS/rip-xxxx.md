---
rip: xxxx
title: Proving unnecessary rollup reorgs
description: A generic approach for proving to the L1 execution environment that a sequencer has unnecessarily equivocated L2 blocks.
author: Ian Norden (@i-norden), Bo Du (@notbdu)
discussions-to: todo
status: Draft
type: Standards Track
category: Core
created: 2024-11-03
---

## Abstract

This proposal describes a simple general means to prove within the L1 execution environment that a sequencer has needlessly equivocated rollup blocks.

This approach will work for any rollup that meets the following requirements:
1. The rollup settles its state roots to the L1 in a trustless manner (e.g. using fault proof system or validity proofs).
2. The rollup maintains within its execution environment a trusted view of the L1 state that the current rollup block derives from (e.g. EIP-4788 or L1BlockInfo). This L1 state is referred to hereon out as the L1 origin of a L2 block.
3. The correctness of the rollup->L1 origin mapping is enforced by the constraints of the fault proof system or validity proofs used to settle rollup state roots to the L1.
4. The sequencer signs preconfirmations to all rollup blocks whose ordering has not yet been finalized through L1 data availability commitments.

We believe the above requirements are basic requirement for any rollup and, as such, this proposal does not add any new constraints on the L2.

## Motivation

Rollups provide three levels of finality:
* FINAL: The transaction data for this rollup block has been submitted to the data availability layer and the transaction containing the commitment to this DA batch has been finalized on the L1.
* SAFE: The transaction data for this rollup block has been submitted to a data availability layer and a transaction containing the commitment to this DA batch has been submitted on the L1 but has not been finalized.
* UNSAFE: The transaction data for this rollup block has not been submitted to the data availability layer or it has been submitted but the transaction committing the DA batch into the L1 has not been submitted to the L1.

Once SAFE finality has been achieved the only mechanism by which the sequencer can reorg the committed segment of blocks is if an L1 reorgs such that the DA commitment transaction is removed.

Once FINAL finality has been achieved there is no mechanism by which the sequencer can reorg the committed segment of blocks.

UNSAFE finality is often referred to as a “preconfirmation”, at this level the only guarantee that blocks will not be reordered before the ordering is committed into the L1 is a promise made by the sequencer.

Rollup sequencers can attest to the ordering of transactions in this UNSAFE region by gossiping signed payloads corresponding to preconfirmed states. It is possible that a sequencer may be _required_ to reorg these preconfirmed rollup blocks in the event that the L1 state these blocks derive from becomes non-canonical due to a L1 reorg.

Currently, there is no general mechanism described by which a user can prove a sequencer reneged on their preconfirmation ordering guarantees when doing so was not necessitated by a reorg of a L1. Herein we describe such a mechanism that leverages the L1 origin view committed into a rollup to allow users to prove within a contract deployed inside the L1 execution environment that a sequencer reorged a preconfirmed rollup block- equivocating at a given rollup block height- when an L1 reorg did not require such an action.

This functionality extends to SAFE reorgs as well. If an L1 reorg removes a DA commitment the sequencer does not *need* to and so should not reorder the contents of the DA batch _unless_ the reorg additionally impacted the L1 origins of rollup blocks in that batch. If the batch origins were not impacted by the L1 reorg that removed the batch commitment, then the same batch commitment can- and should- simply be resubmitted to the L1.

This capability is useful in providing a means of recompense in the event of a sequencer unnecessarily reordering UNSAFE rollup blocks in a manner that negatively impacts users. When used in combination with mechanisms that slash sequencers or payout users in this event, this provides a means of reducing reliance on “good” sequencer behavior. This document only describes the mechanism for proving unnecessary equivocation within the L1 execution environment, precisely how this proof is processed to affect a downstream outcome and the nature of the outcome (e.g. sequencer slashing, insurance payout, etc) is not specified here.


## Specification

Proving equivocation between a signed ExecutionPayloadEnvelope and a settled L2Output is straightforward assuming a fault proof system (or validity proofs) are in place.
1. User submits the valid L2Output at the equivocated block height to the proof system
2. Once it is verified by the proof system…
3. User submits a signed ExecutionPayloadEnvelope that has the same block height but a different blockhash
4. Contract checks that
   1. ExecutionPayloadEnvelope.BlockHash ≠ L2Output.BlockHash
   2. ExecutionPayloadEnvelope.BlockHeight == L2Output.BlockHeight
   
In order to only punish equivocation which was unnecessary we need to not punish equivocation if it is because the L1Origin changed (L1 reorged) and necessitated a reorg of the unsafe L2 block.

To accomplish this we just need to add a step to our contract so that we check that the L1Origin is the same. The steps become:
1. User submits the valid L2Output at the equivocated block height to the proof system
2. Once it is verified by the proof system…
3. User submits a signed ExecutionPayloadEnvelope that has the same block height but a different blockhash
4. Contract checks that
   1. ExecutionPayloadEnvelope.BlockHash ≠ L2Output.BlockHash
   2. ExecutionPayloadEnvelope.BlockHeight == L2Output.BlockHeight
   3. ExecutionPayloadEnvelope.L1Origin == L2Output.L1Origin

IFF the above are all true then the sequencer reorged a L2 block when it did not need to. We can then have arbitrary downstream mechanisms to slash/punish them or reward/recoup losses to the affected parties.
Slashing/punishing is easier as the above establishes- inside the L1 execution environment- an objective attribution of fault linked to the sequencer, but it is more difficult to recompense affected parties as we have not established an objective link to them yet.

### Pseudocode

## Security Considerations

Implementers must make sure that the proof verification contract on the L1 is bug free.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).