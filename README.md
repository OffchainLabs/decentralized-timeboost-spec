# Decentralized time boost sequencer

**Goals:**

- good latency for retail transactions and for arbitrage
- transaction submitters are able to protect their transactions from front-running and sandwiching, assuming the committee is threshold-honest
- provide a decentralized version of the priority controller auction approach to time boost
  - a priority controller, who is chosen by an external auction (or similar) process, can get transactions included faster than anyone else, so it can win any arbitrage races

**Assumptions:**

- sequencer committee of *N* nodes, of which at most *F < N/3* are byzantine malicious
  - members are chosen so that the community trusts at least N-F members are honest
- each committee member has a clock with one-second granularity, and the clocks of honest nodes are roughly in sync
- time is divided into epochs of 60 seconds each, with epoch *i* starting at timestamp *60i* and lasting until timestamp *60i+59*, The function $\mathrm{epoch}(t)$ returns the epoch number to which timestamp $t$ belongs.
- before each epoch, an external mechanism designates one party (by its public key) as the express-lane controller for that epoch

**Submitting transactions**

Submitters are encouraged to send their transactions to every member of the committee, to ensure the earliest inclusion in the chain. Any transaction received by at least *F+1* honest committee members is guaranteed to be included eventually. (For purposes of this guarantee, “included” means that at some point in the chain’s canonical history, the transaction is either included in the chain or correctly rejected as invalid.)

Transactions can optionally be encrypted as a protection against front-running by malicious committee members.

Many users are expected to rely on intermediaries to encrypt their transactions and multicast them to the committee. Users’ legacy wallets would likely submit transactions via such an intermediary.

Members may discard any received transaction that is clearly invalid and cannot become valid within the next minute. They may also discard any received transaction after one minute if that transaction has not progressed in the protocol.

**Background: BFT consensus sub-protocol**

The protocol relies on a consensus sub-protocol, which implements the following functionality. 

In each round, every honest member submits a message. If the round succeeds, all honest members receive the same result, which consists of messages submitted in the round by *N-F* of the members. Or the round can report failure.

Multiple rounds can be in progress at the same time–a round can start before previous rounds have completed–but the rounds must start in order and complete in order. 

The details of the consensus sub-protocol are not specified here.

**Background: threshold decryption**

The protocol relies on a CCA-secure threshold encryption scheme that is not specified here. Transactions and bundles can be submitted in encrypted form, and these will be decrypted jointly by the members after their position in the sequencer’s output can no longer be manipulated.

Each member of the sequencer committee holds a key share, and *F+1* decryption shares are needed to decrypt an encrypted item. 

* This ensures that if an encrypted message is decrypted, at least one honest member must have participated in the decryption.

**Background: timeboost**

Within each one-minute *priority epoch*, there is a single *priority controller* address which is announced by an external protocol that is specified elsewhere. Each priority epoch starts at a predetermined time that is known to everyone, so given a timestamp anyone can compute which priority epoch it belongs to.

The address of the next epoch’s priority controller will be announced at least 15 seconds before the epoch begins, by a valid transaction to a special precompile. Members must always know the address of the current priority controller.

The priority controller can submit *priority bundles* which will be given favorable treatment in the sequencing. (Some bundles may be singletons containing only one transaction.) The time advantage of priority bundles is intended to guarantee that the priority controller can win any races to capture arbitrage value.

A priority bundle is submitted by wrapping the bundle into a transaction sent to a special reserved address called *PriorityAddr*. Transactions sent to *PriorityAddr* are consumed by sequencer committee members, and are never included directly in the chain. To process a transaction to *PriorityAddr*, a sequencer committee member does the following:

- Parse the nonce field of the transaction into a 128-bit epoch number and a 128-bit sequence number
  - Sequence numbers restart from zero in each new epoch, and should increment consecutively within each epoch. The protocol will ensure that bundles within an epoch are included in the chain in sequence-number order.
- If the epoch number does not equal the current epoch number or the current epoch number plus 1, ignore the transaction.
- Otherwise, if the transaction has not been signed by the priority controller address for its epoch number, ignore the transaction.
- Otherwise, unpack the transaction to construct a priority bundle, with the transaction’s calldata becoming the bundle's contents (which might be encrypted), and the extracted epoch number and sequence number becoming the epoch number and sequence number of the bundle. 

Transactions submitted to addresses other than *PriorityAddr* are called *non-priority* transactions and have the normal transaction semantics.

**Configuration and key management**

The protocol's configuration is managed and stored by a smart contract on the parent chain. The configuration consists of:

* the number of committee members, N
* the (assumed) maximum number of malicious members, F, which must satisfy 3F < N,
* a public signature verification key for each committee member, in a signature scheme such as BLS which supports signature aggregation,
* all public key and configuration information for the threshold encryption scheme,
* an additional address (in addition to the chain owner) that can exercise the owner's privileges in this configuration contract; this can be set to address zero if not needed

The chain's owner, or the specified additional address, is allowed to:

* change the additional address only, with immediate effect, or

* schedule a change to the entire configuration, which must be scheduled for a future timestamp.

The smart contract will allow anyone to read the full configuration (or parts of it, for convenience), and to see any scheduled configuration change (both the scheduled timestamp and the full contents of the change).

Information about how to connect to committee members must be available to everyone out-of-band, e.g., published as a json string at a well-known URL.

**Overview of the sequencing protocol**

The protocol operates in rounds, which are not the same as priority epochs. There is no fixed ratio or synchronization between rounds and priority epochs.

Each round operates in three phases, and results in pushing a sequence of transactions into a queue which will be consumed by the block-building engine.

- The *inclusion* phase produces a consensus inclusion list, which contains a set of transactions and priority bundles, and some metadata. Each transaction (or bundle) in an inclusion list may or may not be encrypted under the threshold encryption scheme.
- The *decryption* phase takes the consensus inclusion list produced by the inclusion phase, and threshold-decrypts any encrypted transactions or bundles in the list.
- The *ordering* phase takes the decrypted inclusion list produced by the decryption phase, and produces an ordered sequence of timestamped transactions, which are pushed into a queue that will be consumed by the block-building engine. 

The *block-building* *engine* consumes the queue of transactions produced by the ordering phases of all rounds, and produces the final sequence of blocks by executing the transactions, filtering out invalid ones, and packing them into blocks. This process is deterministic, so all honest members can do it separately in parallel, and they are guaranteed to get the same result. At the end they sign the resulting blocks, and these signatures can be aggregated into quorum certificates on the blocks.

The inclusion phase operates on its own cadence, starting the next consensus round without waiting for the other phases to finish processing the previous round’s transaction list. The decryption and ordering phases can be pipelined, subject to the timing constraints described below. The block-building engine executes independently, consuming the transactions enqueued by the ordering phases of rounds.

**Inclusion phase**

The inclusion phase is specified in [a separate document](inclusion.md).

**Decryption phase**

Some or all of the transactions and bundles in the consensus inclusion list may be encrypted. The decryption phase will decrypt them. This phase consumes the results of the inclusion phase, which will be the same for all honest members, and all honest members will produce the same output for this phase.

The decryption phases of multiple rounds can be done concurrently.

If any of the transactions or bundles in the inclusion list are encrypted, the committee member multicasts its decryption shares for the encrypted items. It then awaits the arrival of decryption shares from other committee members. As soon as it has received *F+1* decryption shares for an encrypted item (including its own share), it can use those shares to decrypt the item.

(Note that decryption of a non-priority transaction could lead to duplicate copies of the same transaction, because the result of decryption could be identical to a transaction that is already present. This cannot happen for priority bundles, nor for delayed inbox messages, because they have sequence numbers that have already been de-duplicated.)

As soon as the member has decrypted all of the encrypted items in the inclusion list (or immediately, if there were no encrypted items), it first de-duplicates the set of non-priority transactions by removing all but one instance of any transaction that is duplicated in the set, and then the member passes the de-duplicated inclusion list, tagged with the round number, to the next, ordering phase. 

<u>State and recovery in the decryption phase</u>

The decryption phase is stateless. No state needs to be remembered from one round to the next. So a member can participate in the decryption phase of any round, if it knows the correct inclusion list result for that round.

**Ordering phase**

Each honest committee member runs a separate instance of the ordering phase. This phase is deterministic, and consumes inputs that will be the same for all honest members, so it will produce the same sequence of outputs for every honest member.

The ordering phase consumes the inclusion lists produced by the decryption phase, and produces an ordered sequence of timestamped transactions, which are pushed into a queue that will be consumed by the block building engine. 

The inclusion lists from multiple rounds can be processed concurrently, however the results must be emitted from the ordering phase in round order.

An honest member sorts the transactions and bundles in the inclusion list as follows:

- order all priority transactions or bundles ahead of all non-priority transactions
- among the priority transactions or bundles, order according to the sequence number given by the priority controller, then break up each bundle into its constituent transactions (by SSZ-decoding the payload as a List[List, 1048576], 1024]), maintaining the order from the bundle
- among the non-priority transactions, order according to Hash(seed || sender address), breaking remaining ties by nonce (increasing), breaking remaining ties by Hash(contents)
- among the delayed-inbox transactions, order by arrival order in the L1 inbox

At this point, the ordering phase waits until the ordering phases of all previous rounds have completed, and then:

- Pushes the transactions in order, and timestamped with the consensus timestamp from their inclusion list, into the input queue of the block building engine.
- Push the indices in the consensus delayed inbox index sequence,  in increasing order, into the input queue of the block building engine. (Note that the interval might be empty.)

This completes the ordering phase for the round.

<u>State and recovery for the ordering phase</u>

Each round of the ordering phase is self-contained, and there is no state that needs to be remembered from one round to the next. So a member can execute the ordering phase for any round for which it knows the correct result of the decryption phase.

**Block building engine**

Each honest committee member runs an instance of the block building engine.

The job of the block building engine is to consume timestamped transactions from its input queue and use them to build blocks. This uses the same block-building logic already included in “back end” of the current Arbitrum sequencer, which executes the transactions, filters out transactions that are invalid or unfunded, and packs the transactions into blocks. 

This would use the ProduceBlockAdvanced function in the Nitro code, or something similar. However, this operation must be deterministic, which probably requires updates to the existing code, for example to use the timestamps on transactions rather than reading the current clock.

This phase may optionally (but identically for all members) include a nonce reordering cache, which remembers transactions that would generate nonce-too-large errors, in the hope that the missing nonce will arrive soon, allowing the erroring transaction to be re-sequenced successfully. (This corrects for out-of-order arrival of transactions.) The cache operates as follows:

* Any transaction that cannot execute successfully because of a nonce-too-large error is added to the cache.
* The cache holds up to 64 entries. If 64 entries are present and one needs to be added, the entry that was inserted earliest is discarded.
* A transaction with timestamp `t` is discarded from the cache the first time a transaction timestamped `t+30 seconds` or greater is ready to execute.
* If a transaction with sender S and nonce N is executed successfully, the cache is checked for a transaction with sender S and nonce N+1. If such a transaction is in the cache, it is removed from the cache and executed immediately, next in the transaction sequence, as if it had arrived immediately after the (S, N) transaction. (Its timestamp is adjusted accordingly, to equal the timestamp of the (S, N) transaction.)
  * Note that this step may execute multiple times. For example, perhaps transactions from S with nonces 6, 7, and 8 are in the cache. If a transaction from S with nonce 5 executes, then all three transactions (6, 7, and 8) will be executed, in order, after 5; and all three will be removed from the cache.

All honest members will see the same sequence of transactions in their local input queue, and this phase is deterministic, so honest members can do this phase independently and asynchronously, and they are guaranteed to produce the same sequence of blocks.

Honest members sign the hashes of the blocks they produce, and multicast their signature shares. *F+1* signature shares can be aggregated to make a quorum certificate for the block. (This is sufficient because at least one of the signers is honest, and therefore an honest member must have signed the blocks that result from correct processing of the unique result of the inclusion round.)

<u>State and recovery for the block building engine</u>

[TO DO. The main issue here is how to get the nonce re-ordering cache into sync.]
