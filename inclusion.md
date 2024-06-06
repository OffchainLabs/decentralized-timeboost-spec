
**Inclusion phase**


The inclusion phase is specified as a set of properties that the inclusion sub-protocol must satisfy.  This is followed by a description of a reference implementation strategy, which satisfies those properties.  The reference is included for its explanatory value; implementations can use a different strategy as long as the required properties are satisfied.

**Inclusion phase: required properties**

Assumptions:

1. Each member has a local clock, which is non-decreasing. If the universal time (i.e., the true, correct time) is $t$ and an honest member $m$ has local clock $t_m$, then $|t_m-t| \le d$.  
 - NOTE: This implies that the clocks of two honest members cannot differ by more than $2d$.
- The L1 delayed inbox (a contract on the L1 chain) has a finality number, which is non-decreasing.
- Each member has a view of the delayed inbox finality number, which satisfies:
  - safety: the member’s view is $\le$ the true number
  - liveness: if the true number is $i$, then the member’s view will eventually be $\ge i$

Round structure:

1. The inclusion phase proceeds in rounds
- Each member locally starts rounds in successive order
- There is a well defined point in time at which any member $m$ starts any given round $R$. We denote by $\mathrm{start}(m,R)$ this start time *with one second resolution, as measured on $m$'s local clock*.
- An implementation should make a best effort to start rounds at a rate of at least once every 250 milliseconds.

 

The result of a round is either FAILURE or a block that contains:

1. a round number $R$
- a predecessor round number $P$
- a consensus timestamp $T_R$
- a pair of delayed inbox finality numbers, $I_{R,\mathrm{first}}$ and $I_{R,\mathrm{next}}$
	- these represent a consecutive sequence of delayed inbox messages, with numbers $[I_{R,\mathrm{first}}, I_{R,\mathrm{next}})$
- an unordered set $N_R$ of non-priority transactions. A subset of these may be encrypted. If a non-priority transaction is encrypted, then the entire transaction is encrypted as a single ciphertext.
- an ordered set $B_R$ of priority bundles. A subset of these may be encrypted. If a priority bundle is encrypted, then the payload (i.e. the contents of all transactions contained in the bundle) is encrypted as a single ciphertext, with the other fields of the bundle (including epoch number, sequence number, and signature) remaining in plaintext.

The result of round number zero is predetermined and is considered to have been committed by all parties. It has:
- $R = 0$
- $P = 0$
- $T_R = 0$
- $I_{0,\mathrm{first}} = I_{0,\mathrm{next}} = I_\mathrm{init}$, where $I_\mathrm{init}$ is set administratively
  - $i_\mathrm{init}$ is the first delayed inbox message that needs to be sequenced by this protocol instance

- $N_R = \emptyset$
- $B_R = \emptyset$

Standard consensus properties:

1. If an honest member has committed a result for round $R > 0$, then it has committed a result for round $R-1$.
- If two honest members both commit a result for round $R$, then they commit the same result.
- A result will eventually be committed for each round $R > 0$
- Under well-defined and standard "optimistic" conditions (e.g., intervals of synchrony), consensus will commit blocks (rather than FAILURE).

An additional property we require is this:  

1. For a round $R > 0$, no honest member commits a result for round $R$ or starts round $R+1$ before at least $N-2F$ honest members have started round $R$.

If a non-FAILURE result has been committed by an honest member for a round $R > 0$, then at the time of commitment and all later times, the result satisfies these properties:


1. $P < R$
- for all $i$ such that $P < i < R$: the member committed FAILURE as the result of round $i$
- $T_R \ge \max(T_P, \mathrm{start}(m,R))$ for some honest member $m$
- $T_R \le \max(T_P, \mathrm{start}(m',R))$ for some honest member $m'$
- $I_\mathrm{R,\mathrm{first}} = I_{P,\mathrm{next}}$
  - NOTE: This ensures that there are no gaps between the $I$ sequences from consecutive rounds
- $I_{R,\mathrm{first}} \le I_{R,\mathrm{next}} \le I$ where $I$ is the true L1 delayed inbox finality number
- the bundles in $B_R$ are sorted in increasing order of sequence number
- If a bundle $b \in B_R$ and $b$ has epoch $e_b$ and sequence number $s_b$, then:
  1.  $e_b = \mathrm{epoch}(T_R)$
  - for every round $R' < R$, and every bundle $b' \in B_{R'}$, either $e_{b'} < e_b$ or ($e_{b'} = e_b$ and $s_{b'} < s_b$)
    - Across all rounds, bundles are in lexicographic order by (epoch number, sequence number)
  - if $s_b \ne 0$, then there is some $R' \le R$ and  $b' \in B_{R'}$ such that $e_{b'} = e_b$ and $s_{b'} = s_b-1$ 
    	- NOTE: This says that there are no gaps in the sequence numbers included within an epoch
- Let $n$ be a non-priority transaction that is included in the result of round $R$, and $b$ be a priority bundle that is included in the result of round $R' > R$. Let $b$ have epoch number $e_b$ and sequence number $s_b$. Let $\tau$ be the (universal) time at when $n$ first arrived at any honest member. Then there is some $s \le s_b$ such that bundles with epoch $e_b$ and sequence number $s$ arrived at fewer than $F+1$ honest members before (universal) time $\tau+250\ \mathrm{milliseconds}$.
	- NOTE: This says that priority bundles indeed take priority over non-priority transactions

Inclusion guarantees:

1. Suppose a block is committed in round $R > 0$. Suppose that 
there exists a non-priorty transaction $n$ such that each honest party receives $n$ at least 250 milliseconds before it starts round $R$. Then $n$ should be included in round $R$ (or an earlier round). 
	- This is a "best effort" requirement, in that it may fail to attain in periods of unexpectedly high demand
- Suppose a block is committed in round $R$. Suppose that there exists an epoch number $e$ such that $\mathrm{epoch}(\mathrm{start}(m,R))=e$ for each honest member $m$. Let $s$ be a sequence number. Assume that either $s=0$ or a bundle with epoch $e$ and sequence number $s-1$ is included in round $R$ or an earlier round. Assume that there is a set $S$ of $F+1$ honest members such that each $m \in S$ receives a bundle with epoch $e$ and sequence number $s$ before it starts round $R$. Then some bundle with epoch $e$ and sequence number $s$ is included in round $R$ (or an earlier round).
	- Note that the assumption "$\mathrm{epoch}(\mathrm{start}(m,R))=e$ for each honest member $m$" implies that $\mathrm{epoch}(T_R) = e$
	- This assumption also implies that for each $m \in S$ will also view the current epoch number as $e$ when it starts round $R$
	- Alternatively, we could make the weaker assumption that $\mathrm{epoch}(T_R) = e$ and that $\mathrm{epoch}(\mathrm{start}(m,R))=e$ for all $m \in S$
	- Note that different members might have received different bundles with the same (epoch, sequence number) values. In that case the guarantee ensures that one of them will be included
	- This is also a "best effort" requirement, but should only fail under extreme conditions


**Inclusion phase: reference implementation strategy**

This section describes a reference implementation strategy for the inclusion phase. This strategy satisfies all of the requirements, but implementations are free to use a different strategy if it also satisfies the requirements.

In each round of the consensus sub-protocol, each committee member submits a candidate list of transactions. The consensus round’s result is a list of $N-F$ of the candidate lists submitted for that round (or no result, if the consensus round fails).

When a member $m$ constructs its *candidate list* as its input to a consensus round $R$, it includes all of the following in its candidate list:

1. $m$'s current timestamp, which defines $\mathrm{start}(m,R)$ 
- $m$'s view of the largest index of any delayed inbox message that has reached finality on the parent chain
- all priority bundle transactions from the priority epoch $e=\mathrm{epoch}(\mathrm{start}(m,R))$ that the member has seen,
- all non-priority transactions that arrived at this member at least 250 milliseconds ago, 
- [to help crashed/restarted nodes recover:] information about (the member’s view of) the latest consensus inclusion list that was produced by a previous round:
  1. Round number
  - Consensus timestamp
  - Consensus priority inbox index
  - Next expected priority bundle sequence number (i.e., the value of K+1 that caused the “S is empty” condition to be reached)

Members should make a reasonable best effort to exclude from their candidate lists any transactions or bundles that have already been part of the consensus inclusion list produced by a previous round. (Failures to do so will reduce efficiency but won’t compromise the safety or liveness of the protocol.)

When the consensus sub-protocol commits a result, all honest members use this consensus result to compute the result of the inclusion phase, called the round’s *inclusion list*, which consists of:

1. The round number
- A consensus timestamp, which is the maximum of:
  - the consensus timestamp of the latest successful round, and
  - the median of the timestamps of the candidate lists output by the consensus protocol
  		- **[VICTOR] if we take a median of an even number of integers, we technically get a fraction. What is the rule to convert to an integer? I don't think ut matters, but it should be specified.**
- A consensus priority epoch number, which is computed from the consensus timestamp
- A consensus delayed inbox index, which is the maximum of:
  - the consensus delayed inbox index of the latest successful round, and
  - the median of the delayed inbox indexes of the candidate lists output by the consensus protocol	
- A (possibly empty) sequence of delayed inbox indices, consisting of the interval `(prev, current]` where `prev` is the consensus delayed inbox index of the latest successful round, and `current` is the consensus delayed inbox index of this round.
- Among all priority bundle transactions seen in the consensus output that are tagged with the current consensus epoch number, first discard any that are not from the current consensus epoch and any that are not properly signed by the priority controller for the current epoch. Then include those that are designated as included by this procedure:
  - Let K be the largest sequence number of any bundle from the current consensus epoch number that has been included by a previous successful round’s invocation of this procedure, or -1 if there is no such bundle
  - Loop:
    - Let S be the set of bundles from the current epoch with sequence number K+1
    - If S is empty, exit
    - Otherwise include the contents (calldata) of the member of S with smallest hash, increment K, and continue
- All non-priority transactions that appeared in at least $F+1$ of the candidate lists output by the consensus round, and for each of the previous 8 rounds, did not appear in at least $F+1$ of the candidate lists output by that previous round

If a consensus round fails, it produces no inclusion list, and the later phases of the protocol are not executed.

The resulting list is called the *consensus inclusion list* for the round. All honest committee members will output the same consensus inclusion list. The output, tagged with the round number, is passed to the decryption phase.

If a non-priority transaction appears in at least one of the candidate lists, but fewer than $F+1$, then the transaction will not be included in the consensus inclusion list. But honest members who have not previously received such a transaction must treat such a transaction as received. 

Notes:

1. Some messages that were included in the submitted candidate lists may not qualify for inclusion in the round’s consensus inclusion list. An honest committee member who has not yet seen such a message will now have seen it, so will include it in later rounds’ inputs. Eventually such a message will make it into some round’s consensus inclusion list.
- Members must keep the following state between rounds: the number of the last successful round; the consensus timestamp of that round; the consensus delayed inbox index of that round; the next expected priority bundle sequence number at the end of that round (i.e. the value of K+1 that caused the “S is empty” condition). This information is included in the input, to help crashed/restarted members get back into sync.

<u>State management and recovery after a crash</u>

Members must keep the following state between rounds: 

- the number of the last round that successfully completed; 
- the consensus timestamp of that round; 
- the consensus delayed inbox index of that round; 
- the next expected priority bundle sequence number at the end of that round (i.e. the value of K+1 that caused the “S is empty” condition). 
- hashes of all non-priority transactions that, in any of the previous 8 rounds, were seen in at least F+1 candidate lists produced by the consensus protocol for that round

A member who knows this information can safely compute the consensus inclusion list of the next round, and can safely update its state for that next round. 

Members include their latest state information in the input, to help crashed/restarted members get back into sync.

If a member is recovering from a crash, or is newly joining the committee, it will not initially know the state, so it must initially participate only passively in the protocol: not submitting an input; casting votes in the core consensus protocol; not producing a local version of the consensus inclusion list (because it cannot be guaranteed correct); and not participating in later phases of the protocol.

Such a member should record the results of consensus rounds. To get its state into synchronization with other honest nodes, it must do the following:

* observe at least one successful round, so it can synchronize its view of the last successful round number
* observe at least 8 rounds, so it can synchronize its view of the list of hashes from previous rounds,
* if the first successful round it observes is round R, observe until it has seen latest-inclusion-list information from at least F+1 members, that specify round number at least R-1 in each case (but can differ across the F+1 members), which together with the observed rounds, is sufficient to synchronize the consensus timestamp and consensus delayed inbox index for the latest successful round.

After doing these things, the member will be in sync and can start participating actively in future rounds of the protocol, including computing the consensus inclusion list and passing on new information to later rounds of the protocol.

