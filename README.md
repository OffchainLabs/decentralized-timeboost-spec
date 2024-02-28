# Decentralized express-lane time boost sequencer

**Goals:**

- good latency for retail transactions and for arbitrage
- transaction submitters are able to protect their transactions from front-running and sandwiching, assuming the committee is threshold-honest
- provide a decentralized version of the express-lane lottery approach to time boost
  - an express-lane controller, who is chosen by an external auction (or similar) process, can get transactions included faster than anyone else, so it can win any arbitrage races

**Assumptions:**

- sequencer committee of *N* nodes, of which at most *F < N/3* are byzantine malicious
  - members are chosen so that the community trusts at least N-F members are honest
- each committee member has a clock with one-second granularity, and the clocks of honest nodes are roughly in sync
- time is divided into epochs of 60 seconds each, with epoch *i* starting at timestamp *60i* and lasting until timestamp *60i+59*
- before each epoch, an external mechanism designates one party (by its public key) as the express-lane controller for that epoch

**Generally**

- the protocol goes in rounds, where each round makes a block (or no block, if there are no transactions)
- rounds are pipelined to the extent possible

**Requirements**

In the absence of network disruptions, and with *N <= 20* committee members who have good network connectivity but are geographically distributed

- a new round starts every 250 milliseconds
- for each round, from the start of Step 1, until the round’s block is produced and fully signed, no more than 1.5 seconds have elapsed
- in any case, round *i* must complete before round *i+16* is allowed to begin

**Submitting transactions and bundles**

- the express-lane controller submits transaction bundles (”priority bundles”) to all committee members; these are optionally threshold-encrypted, and are tagged (outside the encryption) with an (epoch number, sequence number) pair, where the sequence number is assigned sequentially, starting at zero for each epoch
- other users submit (non-priority) transactions to all committee members; these are optionally threshold-encrypted

**Steps in a round**

<u>At the start of a round</u>, each honest committee member makes a “candidate list” of transactions and priority bundles that it has seen and wants to include in the round

- priority bundles are added to an honest member’s candidate list as soon as they’re seen
  - exception (arrival before epoch starts): if during epoch *i*, a priority bundle tagged with epoch *i+1* arrives, the honest member will buffer the priority bundle until the beginning of epoch *i+1*
  - exception (out of order arrivals): if a priority bundle with sequence number *N+1* arrives at an honest member, but that member has not yet seen sequence number *N*, then *N+1* is buffered by the member until *N* arrives, and *N+1* is treated as if it arrived simultaneously with *N*
- honest members artificially delay non-priority transactions by a fixed delay period *D* (say, 200 milliseconds) before adding them to the candidate list
  - this rule is equivalent to saying that the deadline for inclusion in a round is *D* earlier for non-priority transactions than for priority bundles
  - this is part of the express-lane advantage that priority bundles get—they don’t experience this delay
  - note: if a non-priority transaction is not encrypted, then a malicious committee member that sees it could collude with the express-lane controller to front-run it
- each honest committee member tags its candidate list with a timestamp, with one-second granularity, marking the time when the round began

<u>Step 1</u>: the committee uses an Asynchronous Common Subset (ACS) or similar sub-protocol to select the candidate lists submitted by *N-F* of the committee members;

- every honest member computes the round’s timestamp as the maximum of (a) the median of the timestamps on the selected candidate lists, and (b) the timestamp of the previous round
- every honest member computes the round’s “inclusion list” which consists of all of these:
  - all priority bundles that occur in at least one of the selected candidate lists, provided that they are signed by the express-lane controller of the epoch (as determined by the timestamp) and have the expected (epoch number, sequence number)
    - exception: if multiple different bundles with the same sequence number meet this criterion, none of them are included (because the express-lane controller has equivocated)
  - all non-priority transactions that occur in at least $F+1$ of the selected candidate lists

*Guarantees after step 1:*

- *(1a) all honest members have the same timestamp*
- *(1b) the timestamps of consecutive rounds are non-decreasing*
- *(1c) the round’s timestamp is greater than or equal to the timestamp assigned by at least one honest member in this round, and less than or equal to the timestamp assigned by at least one honest member in this round or some previous round*
- *(1d) all honest members have the same inclusion list*
- *(1e) if a priority bundle was received by at least F+1 honest members before the round started, and there exists no other bundle signed by the express-lane controller with the same sequence number, the bundle is in the consensus inclusion list*
- *(1f) if a non-priority transaction was received by at least 2F+1 honest members at least D before the round started, it is in the consensus inclusion list*
- *(1g) if a non-priority transaction is in the consensus inclusion list, it was received by at least one honest member at least D before the round started*
- Note: if the express-lane controller equivocates by signing multiple bundles with the same sequence number, it might lose the ability to get bundles included. This is intentional. Also, the mechanism that chooses the express-lane controller might be designed to “eject” a controller who is proven to equivocate in this way; if this happens, honest committee members should behave accordingly.*

<u>Step 2</u>: the committee threshold-decrypts any encrypted items in the consensus inclusion list, using a scheme requiring *F+1* decryption shares; and the committee jointly selects a random seed for the round

*Guarantee after step 2:*

- ​	*(2a) an item is decrypted successfully if and only if it is in the consensus inclusion list*

  

<u>Step 3</u>: each honest committee member builds a block (or blocks):

- append to the “block building queue” the transactions in any priority bundles that are in the consensus inclusion list, in sequence number order
- append to the block building queue any non-priority transactions in the consensus inclusion list, sorted into order by Hash(seed || sender address), breaking ties by transaction nonce (increasing)
  - if there are multiple transactions with the same sender address and nonce, keep the one with the smallest Hash(seed || transaction) value and discard the rest
- execute the transactions in the block building queue, in order, to filter out ones that are invalidly formatted, unfunded, or have an unexpected nonce; and to determine how much gas each surviving transaction uses
- pack the surviving transactions, in order, into a block, which is tagged with the round timestamp determined in Step 1
  - if the resulting block would contain no transactions, do not emit a block in this round
  - if no more transactions can be added to the block without exceeding the block gas limit, but the block building queue is not yet empty, then produce the block as-is, and leave the remaining transactions in the block building queue; they will be at the head of the queue when the block is built in Step 3 of the next round of the protocol
    - note that this backlogged condition cannot persist for long, because a block that is full to the block gas limit will double the chain’s basefee, so if the backlog persists, the basefee will escalate very quickly, causing transactions to be discarded for not supplying enough gas funds

*Guarantees after step 3:*

- *(3a) all honest members have generated the same block (or agree that no block has been produced)*
- *(3b) every transaction in the generated block is validly formatted, funded, and has the nonce expected by the execution layer*
- *(3c) every transaction in the consensus inclusion list will eventually be included in a generated block, unless the transaction was invalidly formatted, unfunded, or had a bad nonce*
- *(3d) among the transactions produced by a round of the protocol and eventually included in blocks, every priority transaction is ordered before every non-priority transaction*
- *(3e) among the priority transactions produced by a round of the protocol and eventually included in blocks, those transactions are in the order specified by the priority bundles*
- *(3f) consecutive blocks have non-decreasing timestamps*

<u>Step 4</u>: if Step 3 produced a block, every honest committee member signs the block, and $2F+1$ signatures can be aggregated to form a certificate of consensus
