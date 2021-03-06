---
title: Consensus specification
---
# Consensus Algorithm Specification

This article contains the specification of
[the consensus algorithm](../../glossary.md#consensus) in Exonum.

Consensus algorithm in Exonum is the process of reaching an agreement about the
order of [transactions](../../glossary.md#transaction) and the result of their
execution in presence of [Byzantine faults][wiki_bft] and [partially
synchronous][partial_synchrony] network. While executing the consensus
algorithm,
nodes exchange [consensus messages](../../glossary.md#consensus-message)
authenticated with public-key crypto. These messages are processed via [a
message queue](#message-processing) and determine [transitions of nodes among
states](../../architecture/consensus.md#node-states-overview).
The output of the consensus algorithm
is a sequence of blocks of transactions, which is guaranteed to be identical
for all honest nodes in the network.

!!! tip
    See [algorithm overview](../../architecture/consensus.md#algorithm-overview)
    and [list of assumptions](../../architecture/consensus.md#assumptions)
    for more details.

## Definitions

The definitions from the
[general description of the consensus algorithm](../../architecture/consensus.md)
are used. In particular, +2/3 means more than two thirds of the validators,
and -1/3 means less than one third.

### Epochs

The consensus algorithm occurs in *epochs*. During each epoch nodes can agree
on one of the following types of outcomes:

- Apply a **block of transactions**. The nodes need to agree on the ordering
  of transactions, their effect on the blockchain state and errors that
  occurred during execution. Besides transactions,
  [hooks](../../architecture/services.md#hooks) for active services are executed
  before and after transactions.

- Skip block creation on this epoch, instead approving a **block skip**.
  Block skips are similar to empty blocks, but they do not entail executing
  *any* calls. Thus, the blockchain state is unaffected after a block skip
  (or any number of successive block skips).

An epoch is represented by non-negative integer, which is incremented by one
each time a decision is made by the network. Note that this integer may differ
from the blockchain height; the epoch is always greater or equal to the height.

Applying a block increases the blockchain height by one. The block is permanently
recorded into the node storage. In contrast, applying a block skip does not increase
the blockchain height. The node stores only the latest block skip, overwriting
it if a skip is approved at the same blockchain height and the greater epoch.
If the network approves a normal block, a stored block skip (if any) is erased
by the node.

### Rounds

The consensus algorithm proceeds in rounds for each epoch.
Rounds are numbered from 1 and are not synchronized among nodes.

The onsets of rounds are determined by the following timetable.
The first round starts after committing a normal block or a block skip
at the previous epoch.
The second round starts within `first_round_timeout` interval,
The value of which can be configured via
[consensus configuration](../../architecture/configuration.md#consensus-algorithm-parameters).
All further round timeouts are calculated according to the following formula:

```text
first_round_timeout + (r - 1) * round_timeout_increase
```  

Here, `round_timeout_increase` is 10% of the `first_round_timeout`.

Thus, the duration of rounds gradually increases. This provides the network with
more time every round to make a decision on a new block.

### Pool of Unconfirmed Transactions

Each node has a set of transactions that have not yet been added to the
blockchain. This set is called _pool of unconfirmed transactions_. In general,
the pools of unconfirmed transactions are different for different nodes. If
necessary, the nodes
[can request unknown transactions from other nodes](requests.md).

### Proof-of-Lock

A set of +2/3 `Prevote` messages for the same proposal from the nodes at the
current round and epoch is called _Proof-of-Lock (PoL)_. Nodes
store PoL as a part of the node state. The node can have no more than one
stored PoL.

A PoL is greater than the recorded one (has a higher priority), in the following
cases:

- There is no PoL recorded.
- The recorded PoL corresponds to a proposal with a smaller round.

Thus, PoLs are [partially ordered][partial_ordering]. A node must
replace the stored PoL with a greater PoL if it is collected by the node during
message processing.

## Configuration Parameters

- `max_propose_timeout`  
  Initial proposal timeout after a new block is committed to the blockchain.

- `propose_timeout_threshold`  
  If the amount of transactions in the pool of unconfirmed transactions of a
  node is larger than the `propose_timeout_threshold` value, the node switches
  from `max_propose_timeout` to `min_propose_timeout`, i.e. generates blocks
  faster, and vice versa.

- `min_propose_timeout`
  The proposal timeout for the case when the amount of transactions in the pool
  of unconfirmed transactions is larger than `propose_timeout_threshold` value.

- `first_round_timeout`  
  Interval between the first and the second rounds of the consensus algorithm.
  This parameter is used to estimate timeouts for further consensus rounds.
  The estimation formula is
  `first_round_timeout + (r - 1) * round_timeout_increase`, where
  `round_timeout_increase` is 10% of the `first_round_timeout`.

- `status_timeout`  
  Interval between `Status` message broadcasts.

!!! tip
    These parameters are a part of [the global configuration](../../architecture/configuration.md).
    They can be adjusted with the help of [the supervisor service](../supervisor.md).

## Node State Variables

- `current_height`  
  Current blockchain height (i.e., the number of blocks in it).

- `current_epoch`  
  Current epoch of the consensus algorithm.

- `queued`  
  Queue for consensus messages (`Propose`, `Prevote`, `Precommit`) from a
  future epoch or round.

- `proposes`  
  Hash map with known block proposals.

- `locked_round`  
  Round when the node has [locked](../../architecture/consensus.md#locks)
  on a proposal. 0 if the node is not locked.

- `current_round`  
  Current round (1-based).

- `locked_propose`  
  `Propose` on which the node is locked. May be undefined.

- `state_hash`  
  Hash of the blockchain state.

## Consensus Messages

The consensus algorithm uses the following types of messages:
[`Propose`](../../architecture/consensus.md#propose),
[`Prevote`](../../architecture/consensus.md#prevote),
[`Precommit`](../../architecture/consensus.md#precommit),
[`Status`](../../architecture/consensus.md#status),
[`BlockResponse`](../../architecture/consensus.md#blockresponse).

The following fields are present in all messages:

- `validator_id`  
  Index of a validator in the `validators` list in the global configuration.

- `epoch`  
  Epoch of the consensus algorithm to which the message is related.

- `round`  
  Round to which the message is related.

- `hash`  
  Hash of the message.

`Propose` and `BlockResponse` messages have the following additional field:

- `prev_hash`  
  Hash of the previous block in the blockchain.

`Prevote` messages have the following additional field:

- `locked_round`  
  Round in which the author of the message
  [has locked](../../architecture/consensus.md#locks)
  on the proposal which is referenced by the message.
  If the author is not locked on any proposal, the `locked_round` field is 0.

!!! note
    A node that is locked on a proposal must send `Prevote`s only
    for the proposal it is locked on. Thus, `locked_round` in `Prevote`s sent
    by a node is always equal to `locked_round` from its state.

`Prevote` and `Precommit` messages have the following additional fields:

- `propose_hash`  
  Hash of the `Propose` message corresponding to this message.

`Precommit` messages have the following additional fields:

- `state_hash`  
  Hash of the blockchain state after the execution of all transactions in the
  `Propose` referenced by the `Precommit`.
- `time`  
  Local time of the validator during `Precommit` generation. This field does not
  influence the validity of the `Precommit`; rather, it is used
  by [light clients](../../architecture/clients.md)
  to check whether responses from full nodes correspond to the
  current state of the blockchain.

## Algorithm Stages

The algorithm proceeds in stages, transitions among which are triggered
by [incoming messages](#message-processing) and [timeouts](#timeout-processing).

- [Full proposal](#full-proposal)  
  Occurs when the node gets complete info about some proposal and all the
  transactions from the proposal.
- [Availability of +2/3 `Prevote`s](#availability-of-23-prevotes)  
  Occurs when the node collects +2/3 `Prevote` messages from the same round
  for the same known proposal.
- [Lock](#lock)  
  Occurs when the node replaces [the stored PoL](#proof-of-lock) (or collects
  its first PoL for the `current_epoch`).
- [Commit](#commit)  
  Occurs when the node collects +2/3 `Precommit` messages for the same round for
  the same known proposal. Corresponds to [the Commit node state](../../architecture/consensus.md#node-states-overview).

The steps performed at each stage are described [below](#stage-processing).

## Message Processing

Nodes use a message queue based on [the `tokio` library][tokio-lib] for message
processing. Incoming requests and consensus messages are placed in the queue
when they are received. The same queue is used for processing timeouts. Timeouts
are implemented as messages looped to the node itself.

Messages from the next epoch (i.e., `current_epoch + 1`) or from a future
round are placed into a separate queue (`queued`).

### Deserialization

- Check the message against the
  [serialization format](../../architecture/serialization.md).
- If any problems during deserialization are detected, ignore
  the message as something that a node cannot correctly interpret.
- If verification is successful, proceed to
  [Consensus messages processing](#consensus-messages-processing)
  or [Transaction processing](#transaction-processing), depending
  on whether the message is a transaction.

### Transaction Processing

- If the transaction is already committed, ignore it.
- If the transaction is already in the pool of unconfirmed transactions,
  ignore it.
- Add the transaction to the pool of unconfirmed transactions.
- For every known proposal `propose` where this transaction is included:

    - Exclude the hash of this transaction from the list of unknown transactions
      for the `propose`.
    - If the number of unknown transactions for the proposal becomes zero,
      proceed to [Full proposal](#full-proposal) state for `propose`.

### Consensus Messages Processing

- Do not process the message if it belongs to a future round or epoch. In
  this case:

    - If the message refers to the epoch `current_epoch + 1`, add the
      message to the `queued` queue.
    - If the message is related to a future epoch and updates the knowledge
      of the node about the current epoch of the message author,
      save this information according to [the requests algorithm](requests.md).

- If the message refers to a past epoch, ignore it.
- If the message refers to the current epoch and any round not higher than the
  current one, then:

    - Check that the `validator_id` specified in the message is less than the
      total number of validators.
    - Check the message signature against the public key of the validator with
      the `validator_id` index.

- If verification is successful, proceed to the message processing according to
  its type.

!!! note
    Consensus messages can lead to sending *request(s)* based on the information
    in the message. Requests are used to obtain information unknown to the node,
    but known to its peers. This part of message processing is described in the
    [*Requests*](requests.md) article and is only marginally touched upon here.

### Propose

**Arguments:** `propose`.

- If `propose.hash` is already present in the `proposes`
  hash map (i.e., the message has been processed previously), ignore the
  message.
- Check `propose.prev_hash` correctness.
- Check that the specified validator is the leader for the given round.
- Check that the proposal does not contain previously committed transactions
  (`Propose` messages contain only hashes of transactions, so absence of
  hashes in the table of committed transactions is checked).
- Add the proposal to the `proposes` hash map.
- [Request missing information based on the message](requests.md#receiving-propose).
- If all transactions in the proposal are known, go to
  [Full proposal](#full-proposal).

### Prevote

**Arguments:** `prevote`.

- Add `prevote` to the list of known `Prevote` messages for the given proposal
  in `prevote.round`.
- If:

    - the node has formed +2/3 `Prevote` messages for the same round and
    `propose_hash`
    - `locked_round < prevote.round`
    - the node knows a `Propose` message referenced by this `prevote`
    - the node knows all the transactions from the `Propose`

- Then proceed to [Availability of +2/3 `Prevote`s](#availability-of-23-prevotes)
  for the referenced `Propose` message in `prevote.round`.

- [Request missing information based on the message](requests.md#receiving-prevote).

### Precommit

**Arguments:** `precommit`.

- Add the message to the list of known `Precommit`s for `propose_hash` in this
  round with the given `state_hash`.
- If:

    - the node has formed +2/3 `Precommit`s for the same round, `propose_hash`
      and `state_hash`
    - the node knows the `Propose` referenced by `propose_hash`
    - the node knows all the transactions in this `Propose`

- Then:

    - Execute the proposal, if it has not yet been executed.
    - Check that `state_hash` of the node coincides with the `state_hash`
      in the `Precommit`s. If not, stop working and signal about
      an unrecoverable error.
    - Proceed to [Commit](#commit) for this block.

- Else:

    - [Request missing information based on the message](requests.md#receiving-precommit).

### BlockResponse

**Arguments:** `block`.

!!! note
    `BlockResponse` messages are requested by validators if they see a
    consensus message belonging to a future blockchain height or epoch.
    `BlockResponse` messages are not a part of an ordinary consensus
    message workflow for nodes at the latest epoch.

- Check the `BlockResponse` message:

    - The key in the `to` field must match the key of the node.
    - `block.prev_hash` must match the hash of the latest committed block.
    - If the returned value is a normal block, its height must equal
      the current height of the node. If the returned value is a block skip,
      its height must equal the previous height (i.e., `current_height - 1`)
      and its epoch must be greater than `current_epoch`.
    - The number of `Precommit` messages from different validators
      must be sufficient to reach consensus.
    - All `Precommit` messages must be correct.

- If the checks are successful, then check all transactions in the block for
  correctness. If some transactions are incorrect, stop working and signal about
  an unrecoverable error.
- Execute all transactions. If the hash of the blockchain state after the execution
  diverges from that in the `Block` message, stop working and signal about
  an unrecoverable error.

- Add the block to the blockchain and move to a new epoch. Set the value
  of the variable `locked_round` at the new epoch to `0` .

- [Request missing information based on the message](requests.md#receiving-blockresponse).

## Timeout Processing

### Round Timeout

- If the timeout does not match the current epoch and round, skip further
  timeout processing.
- Add a timeout for the `N`th round with the length
  `first_round_timeout * (1 + (N - 1) * q)`,
  where `q = 0.1` is the relative increase in a timeout after each round.
- Process all messages from `queued` that have become relevant (their round
  and epoch coincide with the current ones).
- If the node has a saved PoL, send a `Prevote` for `locked_propose` in the new
  round, and proceed to
  [Availability of +2/3 `Prevote`s](#availability-of-23-prevotes).
- Else, if the node is a leader, form and send `Propose` and `Prevote`
  messages (after expiration of `propose_timeout`, if the node has just
  moved to a new epoch).

### Status Timeout

- If the epoch of the node has not increased since the timeout was set, then
  broadcast a `Status` message to all peers.
- Add a timeout for the next `Status` broadcast (its length is specified by
  `status_timeout`).

## Stage Processing

### Full Proposal

**Arguments:** `propose`, all transactions in which are known.

- If the node does not have a saved PoL, send a `Prevote` message in the round
  to which the proposal belongs.
- For each round `r` in the interval
  `[max(locked_round + 1, propose.round), current_round]`:

    - If the node has +2/3 `Prevote`s for `Propose` in `r`, then
    proceed to [Availability of +2/3 `Prevote`s](#availability-of-23-prevotes)
    for `propose` in `r`.

- For each round `r` in the interval `[propose.round, current_round]`:

    - If +2/3 `Precommit`s are available for `propose` in `r` and with
      the same `state_hash`, then:

        - Execute the proposal, if it has not yet been executed.
        - Check that the node’s `state_hash` after applying transactions in
          `propose`
          coincides with the `state_hash` in the aforementioned +2/3
          `Precommit`s.
          If not, stop working and signal about an unrecoverable error.
        - Proceed to [Commit](#commit) for this block.

### Availability of +2/3 `Prevote`s

**Arguments:** shared `propose_hash` and `round` of the collected +2/3
`Prevote`s.

- If `locked_round` of the node is less than `prevote.round` and the hash of the
  locked `Propose` message is the same as `propose_hash` in the collected
  `Prevote`s, then proceed to [Lock](#lock) for the latter `Propose` message.

### Lock

- For each round `r` in the interval `[locked_round, current_round]`:

    - If the node has not sent `Prevote` in `r`, send it for
      `locked_propose` with the `locked_round` specified as the `locked_round`
      in the node state.
    - If the node has formed +2/3 `Prevote`s in `r`, then change `locked_round`
      to `current_round`, `locked_propose` to `propose.hash` (`propose`
      corresponds to +2/3 `Prevote`s in `r`).
    - If the node did not send `Prevote` for any other proposal except
      `locked_propose` in subsequent rounds after `locked_round`, then:

        - Execute the proposal, if it has not yet been executed.
        - Send `Precommit` for `locked_propose` in `current_round`.
        - If the node has +2/3 `Precommit`s for the same round with the same
          `propose_hash` and `state_hash`, then proceed to [Commit](#commit).

### Commit

**Arguments:** block or block skip (i.e., a proposal with all known transactions,
and the `state_hash` resulting from the execution of all transactions in the
proposal).

- Store the block.
- Push all the transactions from the block to the table of committed
  transactions.
- Increment `current_epoch`. If the block is a normal block, increment `current_height`.
- Set the value of the variable `locked_round` to `0` at the new epoch.
- Delete all transactions of the committed block from the pool of unconfirmed
  transactions.
- If the node is the leader, form and send `Propose` and `Prevote` messages
  after `propose_timeout` expiration.
- Process all messages from the `queued` that have become relevant (their round
  and epoch coincide with the current ones).
- Add a timeout for the next round.

## Properties

!!! note
    Formal proof of the following properties is proven in
    [a separate white paper][exonum-wp].

!!! warning
    The white paper does not consider block skips. While adding other
    possible consensus outcomes does not influence safety and liveness arguments,
    it could adversely affect chain quality.

If:

- Digital signature scheme used in the algorithm is secure (i.e., signatures
  cannot be forged)
- Network is partially synchronous (i.e., all messages are delivered in finite,
  but *a priori* unknown time)
- Less than 1/3 of validators act Byzantine (i.e., in an arbitrary way,
  including
  being offline, having arbitrary hardware and/or software issues or being
  compromised, possibly in a coordinated effort to break the system)

Then the algorithm described above has the following properties:

- **Safety**  
  If an honest node adds a block to the blockchain, then no other honest node
  can add a different block, confirmed with +2/3 `Precommit` messages, to the
  blockchain at the same epoch.

- **Liveness**  
  At any point in time, a block will be committed eventually by an honest node
  in the future. In other words, the transaction processing won’t stall.

- **Weak form of chain quality**  
  1 block out of any `F + 1` (where `F` is one third of the validators)
  sequentially committed blocks is
  guaranteed to be proposed by non-Byzantine validators. This can provide a
  certain degree of _censorship resistance_ (any correct transaction broadcasted
  to every validator will be committed eventually).

[wiki_bft]: https://en.wikipedia.org/wiki/Byzantine_fault_tolerance
[partial_ordering]: https://en.wikipedia.org/wiki/Partially_ordered_set#Formal_definition
[tokio-lib]: https://tokio.rs/
[partial_synchrony]: http://groups.csail.mit.edu/tds/papers/Lynch/podc84-DLS.pdf
[exonum-wp]: https://bitfury.com/content/downloads/wp_consensus_181227.pdf
