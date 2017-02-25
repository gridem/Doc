# Geo Distributed Consensus

Here is a description of consensus that doesn't require the liveness of all nodes to commit the portion of messages.

## Notions

- **Message**: is a message from the client to commit. Consensus algorithm is responsible to agree on the sequence of messages.
- **Bunch**: subgroup of nodes to agree on the messages. Each node can be a member of several bunches.
- **Sequence number**: each node has its own sequence number. On state changing it increments the number.

## State

Each node has the following state:

```cpp

struct MessageType
{
	Propose,
	Promise,
	Process,
};

struct Bunch
{
	NodesSet nodes;
	int votes = 0;
};

struct ExternalState
{
	list Message messages;
	NodeId id;
};

struct State
{
	list Bunch bunches;
	list ExternalState states;
};

struct Proposal
{
	State state;
};
```

## Algorithm

Pseudocode:

```cpp
constexpr FAIL_NODES=1;
constexpr NODES=FAIL_NODES*2+1;

void onPropose(Proposal proposal)
{
	if (commit)
	{
		remove committed messages from the state
		execute committed messages
		continue handling rest
	}
	if (has proposals only: this and src)
	{
		merge proposals
		calculate number of nodes with the same prefix
		for all prefixes that: nodes.size with that prefix > FAIL_NODES
		{
			// has majority
			create bunch
			update bunch.votes
		}
		broadcast state
	}
	else
	{
		remove bunches that contradicts with the src.state
		if (src has promise)
		{
			if (this has promise)
			{
				verify that src state removes contradicted bunches
			}
			update bunches
			vote for bunches
			if vote has majority => broadcast commit
			else broadcast updated state
		}
		else
		{
			// src doesn't have promise
			// while this has promise

			// votes are updated only for promise

			merge list messages
			broadcast updated state
		}
	}
}
```

## Algorithm 2

Fixed version, simplified and updated.

Sequence of actions on message receiving:

1. Merge sequences from all nodes based on sequence id from the particular node (update if id > previous id or discard it otherwise).
2. Obtain nonexistent messages in the node sequence and push back those new messages.
3. Iterate throw each messages for the current node:
	- if there is majority: take existent promise or create it for that majority message. Vote for it for the current node. If votes have majority => commit.
	- no majority: break the loop and sort the rest of the message (nonmajority messages).
	- try to continue the loop.

## Examples

Consider the following example with 3 nodes:

1. N1: 1[M1] -> *
2. 1[M1] -> N2. Created bunch: B(1,2+). 2[M1 B(1,2+)] -> *
3. 2[M1 B(1,2+)] -> N1. Created bunch: B(1+,2+) => commit

1. N1: 1[M1] -> *
2. 1[M1] -> N2. Created bunch: B(1,2+). 2[M1 B(1,2+)] -> *
3. N3: 3[M3] -> *
4. 3[M3] -> N1. Merging messages: M3,M1. 4[M3,M1] -> *
5. 2[M1 B(1,2+)] -> N1. Created bunch: B(1+,2+) => commit M1 and propose M3

1. N1: 1[M1] -> *
2. 1[M1] -> N2. Created bunch: B(1,2+). 2[M1 B(1,2+)] -> *
3. N3: 3[M3] -> *
4. 3[M3] -> N1. Merging messages: M3,M1. 4[M3,M1] -> *
5. 4[M3,M1] -> N3. Created bunch: B2(1,3+). 5[M3,M1 B2(1,3+)] -> *
6. 5[M3,M1 B2(1,3+)] -> N1. Created bunch: B2(1+,3+) => commit M3,M1

1. N1: 1[M1] -> *
2. 1[M1] -> N2. Created bunch: B(1,2+). 2[M1 B(1,2+)] -> *
3. N3: 3[M3] -> *
4. 2[M1 B(1,2+)] -> N3. Created bunch: B(1,2+,3+) => commit M1 and propose M3

Description:
`2[M1 B(1,2+)] -> *`
- 2 is the event id
- [] contains the event description
- M1 - message from node 1, e.i. from node N1
- B() is a bunch
- B(1,2+) has a bunch with a nodes set N1 and N2 where N2 has been voted
- -> * is a broadcast to everyone
- -> N1 received the message by N1

## Latency Consideration

There are several possibilities:

1. Linearizability. W=O(RTT), R=O(RTT) [W=write, R=read]
2. Sequential consistency. W=O(1), R=O(RTT) or W=O(RTT), R=O(1)
3. Casual consistency. W=O(1), R=O(1)

W=O(RTT), R=O(1) case is obvious. Another option W=O(1), R=O(RTT) can be implemented by postpone (batch) writes and apply reads through consensus.

Another interesting possibility is to optimize linearizable reads: one may consider to use sequence id to read committed writes for specified id.

Based on [A Critique of the CAP Theorem](http://arxiv.org/pdf/1509.05393v2.pdf)

## Algorithm 3

The idea is to use 'phantoms' to increase convergence. Let's consider the following example:

1. N1: 1[1,,] -> *
2. N2: 2[,2,] -> *
3. 1[1,,] -> 2. Union: [1,2 1,]. No quorum => sorting [1,1 2,]. Has quorum => 3[1,1+ 2,] -> *
4. 2[,2,] -> 1. Union: [1 2,2,]. No quorum even with sorting => 4[1 2,2,] -> *
5. 3[1,1+ 2,] -> 1. Union: [1 2,1+ 2,]. Quorum: [1+ 2+,1+ 2,]. Commit: {1}. Send: 5[2+,2,] -> *
6. 4[1 2,2,] -> 2. Union [1 2,1+ 2,]. Quorum: [1 2,1+ 2+,]. Send 6[1 2,1+ 2+,] -> *
7. 5{1}[2+,2,] -> 2. Commit {1,2}
8. 6[1 2,1+ 2+] -> 1. Commit {1,2}

Thus number of round trips: 1.5 (3 waitings for incoming message). Let's reduce this amount to 1 RTT.

We could apply phantoms technique: add fake data after sorting to speedup the convergence. See example:

> Note: looks like the approach doesn't work. The thing is that we assume some messages for some nodes and those nodes may commit and crash. After that we don't know about commitment thus the rest of the node may rearrange the sequence.
> 
> Note 2: need to carefully arrange the messages on majority vote (commit). Because majority can differ from current node's message thus the message is going to be dropped.

## Algorithm 4

The new idea is based on majority groups. The steps are the following:

1. Collect the sequences of messages from all nodes.
2. If the prefix doesn't have the majority => sort of messages and broadcast.
3. If the prefix has a majority => create the group with the nodes that have majority. The group contains the set of nodes belonging to the group, sequence of messages to be committed and set of nodes that has the same sequence of message. Sequence of messages and group set are immutable data within group, set of nodes that has majority for the same sequence of messages is mutable data in the group. Only nodes from group are allowed to change the mutable set of nodes.
4. Once the number of nodes has a majority => commit the sequence of messages.

## Tagging and Preexecution

The idea is to include additional actions to be performed during agreement phase in progress e.g. for:

1. Hash checking.
2. Node liveness hints.

Those actions are performed before the commit phase.

### Possible Solution

On message receiving it generates new messages that modifies/replaces original one with additional information. Thus the nodes are agreed on modified message and every node receives the necessary information.

## Disk Consensus

During the consensus we need to ensure the data durability. For that purpose we introduce additional step for the round: flushing and syncing the data on the disk. Once the majority of nodes acknowledges the disk writing, the node can commit the data.

### Optimization

We could use flushing the data based on reduced number of nodes. To achive the property unforgettable we need to implement the following:

1. Each reduced set of nodes must contain the minimal set of flushed nodes.
2. On recover when all nodes crash we need to have at least N - minimal set + 1.

Thus if minimal set = 1 and N = 3 thus on recover from all crashed nodes we need to have all nodes. At any time the group of nodes must contain the flushed node.

The most useful usecase: 5 nodes and 2 nodes as a minimal flush set (in normal case: 3 nodes is required).

### Reliability

To improve reliability and disk flush performance one may consider the following scheme. Each node may contain 2 separated disk to flush the same sequence of changes (changelog). 3 nodes thus may contain 6 disk, and we require 2 disk to flush the data. Those disk may belong to the same node or different nodes depending on the policy.

This solves reliability when the progress is forgotten in case of failure single the most performant disk.

## Outstanding Features

1. Geo-distributed: geo-RTT = 1 in any case while master-based approach may require 2 geo-RTT.
2. Simplified logic: no any specific roles.
3. Idempotent messages: messages may be repeated arbitrary number of times.
4. Low overall number of messages in case of 3 or 4 nodes. Can be optimized further.
5. More robust under different network failures.
6. Data can be persisted on the final phase, don't need to sync the data initially. Thus it can be batched and throughput can be much better even for `fsync` for the price of latency.
7. Quorum may not be intersected: may use 2 vs 2 for 4 nodes and one of group may win (e.g. that has lowest node id). 
8. Fast convergence due to state transmission.
9. At one step may be committed several messages at once from different nodes. 
10. Simplified group membership changes. Reduction is automatic while adding through the log.
11. Hot switch: the algorithm allows on node disconnected automatically add new node based on trigger mechanism (trigger is invoked on node disconnection).
12. UDP ready: due to idempotent requests the nodes may apply policy to resend packets on loss after timeout/k time.
13. Big chunk replication: the minor algorithm modification allows to do a consensus for large chunks without latency/bandwidth overhead.

## Formal Proof

Algorithm:

> *Lemma 1*. If node `i` committed sequence `S_i` with node group `G_i` at time `tc_i` (tc - commit time) thus any node `j` that belongs to `G_i` after time `tc_i` cannot change `S_j` and `S_j` == `S_i`

Proof steps:

1. From contradiction: let `j` node changed the `S_j` to `S_j'`
2. The added message could be either from group `G_i` or outside the group.
3. Moment `t_j` when the message issued from `j` comes to `i` and committed.
4. At moment `t_j` the state `j` (`S_j` and `G_j`) was the same as committed state `i`. 
5. The transition `S_j` to `S_j'` is possible only when the new message `m_k` was received thus `S_j' = S_j \U m_k`
6. The origin of `m_k` cannot be inside the group `G_i` because otherwise the node `i` received from `i` the sequence `S_k` that includes `m_k`.
7. Thus `m_k` are outside the group `G_i`.
8. Between `k` and `j` there is a path that intersects the group.
9. Let the node `l` is the node from `G_i` that accepts the message with `m_k` from the node outside `G_i`. The message time is `tm_l`.
10. `t_l` is the timestamp where the state is the same as committed state `S_i`, `G_i`.
11. `tm_l` >= `t_l` => false because it cannot accept the messages from outside `G_i`.
12. `tm_l` < `t_l` => false because in that case `S_l` would contain message `m_k`.

Contradiction.

> Theorem 1. If `i` and `j` nodes committed sequences `S_i` and `S_j` then `S_i == S_j`.

Steps by contradiction:

1. If sets of `S_i` and `S_j` are the same then the sequence is the same.
3. Consider `G_i` and `G_j`. If there is an intersection => node `l` exists.
4. For node `l` apply *Lemma 1* => contradiction with lemma.
5. Thus `G_i` and `G_j` doesn't have intersection.
6. Due to majority they must have intersection => contradiction with algorithm.

> *Definition 1*. The node is nonoperational if there is no way to commit the sequence.

> *Definition 2*. The node is operational if it is not nonoperational.

> *Definition 3*. The group is operational if any node within group is operational.

> *Lemma 2*. The committed groups are subsets of each other.

Proof:

1. Each committed group has a majority => 2 committed groups has at least 1 node `k` that belongs to those groups.
2. At some point group of `k` was the same as the first committed group and the second committed group.
3. The transmission in group for `k` only removes the nodes from group. 

> *Theorem 2*. At most one committed group is operational on next step.

Proof:

1. Consider the smallest committed group `G_i`. The group has a majority.
2. Another committed group is a superset of `G_i`: `G_j > G_i`.
3. The nodes in `G_j` may communicate only with `G_j - G_i` nodes that don't have a majority for voting. Thus it's nonoperational.

> *Consequence*. Only the smallest committed group may be operational on next step.

> *Theorem 3*. The node outside any committed group is nonoperational.

Proof:

1. The smallest committed group may be operational. There is no any operational groups.
2. If the node outside any group thus due to Lemma 2 it outside the smallest committed group.
3. Thus it's nonoperational.

> *Consequence*. If the node doesn't belong to the smallest committed group it is nonoperational.

### Liveness

> *Lemma*. The node with state with nonempty set of messages at moment `t` broadcasted the message with that state at the moment `t' < t`.

Several cases:

1. The node that proposes the message. According to algorithm the node broadcasts that message.
2. Any change of message set broadcasts the new message set.

> *Lemma*. Majority set of nodes without network failures have the same set of messages.

Proof:

1. Let's assume that 2 nodes `i` and `j` has different set of messages. And set `i` contains message `m_k` that node `j` doesn't contain.
2. According to lemma node `i` must broadcast state with `m_k` to node `j`.
3. Thus node `j` must contain that message `m_k` => contradiction.

> *Theorem 3*. Nodes do the commit if there is no failed node and there is no message lost.

Proof steps:

1. All nodes has the same set of messages.
2. Let's take any node `i`. Any node within group except `i` must broadcast the same state to `i`.
3. Those messages lead the node `i` to commit.  

#### Liveness Issues

Algorithm has liveness issues. Let's consider example:

1. Node `1` has group `G_1 = {2, 3}`
1. Node `2` has group `G_2 = {1, 3}`
1. Node `3` has group `G_3 = {1, 2}`

In this situation the nodes cannot commit.

#### Livelock Detection

If majority of group cannot change the state thus we may stop current step and restart for the next step

#### Livelock Resolving

Sometimes we cannot understand that the majority of group cannot proceed. We could apply the following approach: broadcast the empty group within state. If we have received the majority of empty groups thus we can close the step and restart new step. The proof is simple because group majority must contain all nodes with the same values.

### Nodes Hot Switching

If node becomes unavailable we can replace it for the new one. Because committed states may contain different groups the nodes must synchronize their groups on next step.

Thus the correct algorithm for hot switching must contain 2 fields: old node group from previous step and new nodes for hot switching. Old nodes use intersection while new nodes use union. The 3-rd field is the current disconnected nodes. Size of disconnected nodes must be < old + new.

### Group Membership Changes

Added nodes must be initiated. Initiation procedure involves the following steps:

1. Commit node send all added nodes the commit message with `S` and `G` and original group.
2. New node wait for majority based on original group.
3. Use intersection from `G` and checks that intersection has majority (intersection should have a majority and must be proved).

--Majority must be preserved in old set and new set simultaneously.-- Actually, initiation prevents from the issues.

The thing is that it's possible to have several commit groups and only the operational commit group is allowed.

> *Lemma*. Obtaining intersection from commit groups from any set of majority committed nodes is the same.

Proof:

1. Majority includes at least one node `i` from the smallest commit group.
2. Because commit groups are subsets thus intersection with `i` node yields the smallest group. 

> *Theorem*. Initiation procedure installs the same group for all operational nodes.

2 parts:
1. Group must be the same without initiation procedure.
2. Group is the same in new nodes and operational nodes.

## Timeoutless Masterless Algorithm

> *Lemma 1*. If node `i` committed sequence `S_i` with node group `G_i` at time `tc_i` (tc - commit time) thus any node `j` that belongs to `G_i` after time `tc_i` cannot change `S_j` and `S_j` == `S_i`

Proof:

1. Assume that there is node `j` that has changed the committed sequence `S_j` at time `t'_j > tc_i` and there is no and node that changed the sequence at time `< t'_j`.
2. Because it has changed the sequence `S_j` at the group `G_i` thus timestamps of nodes from `G_i` must be less then incoming request from node `l`.
3. It means that there is node `k != j` that has timestamp `ts^k_j` on node `j` and message has timestamp `ts^k_l > ts^k_j`.
4. It means that node `k` changes the sequence thus the timestamp had been increased at timestamp `< t'_j`.
5. But we assumed that there is no any node by (1). QED.

## Big Chunk Replication Mechanism

The node `i` proposes the commands to other nodes. The proposal is a big message that contains a batch of commands from different clients.

The proposal is a stream. When another node `j` receives the stream it broadcasts the `PreAccept` message. `PreAccept` message reserves the message for the node on completing accepting the message.

The node may commit if all nodes within group accepted the message.

On node disconnection the group is reduced and if there is accepted node that node must broadcast accepted batch to other nodes by broadcasting. Otherwise if the group doesn't contain node with accepted message (only preaccepted is available) thus the commit takes place without that batch.

If `K` nodes has accepted messages and `N` nodes doesn't have accepted messages thus the nodes `K` may deterministically decide to spread the knowledge by:

1. Sort list of `K` and `N` nodes by `node_id`.
2. For each node destination `n` from `N` choose source node from `K` by applying `mod K`.
3. Thus each node from `K` may have number of destination nodes from `0...int(N/K)+1`.

The optimal time must be equal to round trip time. Thus the chunk `size ~= RTT*bandwidth / factor`, where `factor > 1`.

### Optimization

In case of node failure the probable situation is that the rest of the nodes have preaccepted state only. Those nodes may decide to replicate already downloaded blocks of chunk by using e.g. maximum number of chunks from the node.

During group reduction the nodes exchanges the data about number of blocks and deterministically decide how to proceed that data.

## Quorum Intersecion

Common approach is to have a majority by quorums intersection. It's not required actually. The nodes may be split in equal parts without loosing consistency. For that case we need to tie the partitions by choosing the part with  e.g. the lowest node id.

Proof:

1. In theorem before let's assume that we have 2 different commits.
2. Those commits either have node intersection (was proved) or not.
3. But if the parts have the same sizes thus one of the part cannot commit => contradiction.

### Fast Node Optimization

Let's suppose we have 4 nodes and fastest node with the minimal node id. In that case using *Quorum Intersection* consideration and timeoutless algorithm we may have the following fast commit ways: any node except the fastest may commit by agreed only with the fastest node (2 nodes in the commit group). Fast node may commit with any other node. On fast node failure we fall done to 3 nodes commitment.

Thus we could use the following deployment scheme:

- Place the fastest node on the minimal distance from other 3 nodes. It's very usable for cross-DC deployment.

## Problem Statement

Motivation:

1. Reduce latency up to 1 geo-RTT - the smallest possible latency.
2. Consensus based on metadata instead of data. For data we could use asynchronous data transferring and enable enhanced protocol transmission like torrent etc.

Outstanding feature *downtimeless*:

1. Soft update/upgrade.
2. Nodes group reconfiguration.
3. Dead node hot replacement.

Idea:

1. Phase 1: broadcast offsets for the queue that producers belongs only to the same DC.
2. Accept offsets from all nodes. Check offsets from current DC and choose the minimal one. Broadcast it.
3. If equal => commit, otherwise goto 2.

Either `2 * 0.5 RTT` or `3 * 0.5RTT`.

> *Finetuning.* On phase one issue not the current offset, but delta like 80%. May be tuned based on previous commit and data lag.

> *Disk flushing.* Phase 2 should broadcast 2 messages: right after receiving all requests (as before) and right after flush the knowledge about receiving all requests. It increases overall latency by `fsync` time which is negligible.

We should proove that disk flushing allows to restore the consistent committed state after all nodes (or majority nodes) have been crashed. The invariant must be preserved: if the node commits the message thus after restoring we should able to understand and commit the same commit as it was before based on any majority knowledge only.

### Disk

To store meta information and to reduce `fsync` latency SSD disks should be used.

Modified algorithm including data and metadata transferring and flushing:

1. Accepts element `e_i,j` from the producer `i` with producer offset `j` by acceptor from the same DC.
2. Acceptor performs in parallel:
	1. Store and flush element the element `e_i,j`
	2. Broadcast `e_i,j` to the other acceptors.
	3. Send `i,j` to proposer.
	4. On element flush send `i,j'f` to the proposer where `'f` means flushed.
4. On 2.3 initiate consensus:
	1. Phase I: broadcast other proposers the map of `{i,j}` for the current DC where `{i,j}` is a map of `i,j`.
	2. Phase II: on receiving `{i,j}_d` from different DC marked by `d` do the following:
		- combine all `{i,j}_d` for all `d` from DC including itself, the combination is the minimum of all `j` for specific producer `i` for all `d`, the result is just `{i,j}`;
		- do in parallel:
			- flush the result `{i,j}` only if the result is the same as received from other DC nodes meaning that others received `e_{i,j}` elements;
			- broadcast the result `{i,j}`;
	3. Phase II': on receiving flushed data and metadata both `{i,j}'f` and all `e_{i,j}'f` broadcast `{i,j}'f` for the current DC.
	4. Phase III: on receiving flushed the same data from all nodes: commit the meta.

Delay:

0 -> Phase I: local network: 0
Phase I -> Phase II: +0.5 RTT = 0.5 RTT
Phase II -> Phase II': +flush = 0.5 RTT + 1 flush
Phase II' -> Phase III: +0.5 RTT = 1 RTT + 1 flush

Simplified Algorithm:

1. Accepts element `e_i,j` from the producer `i` with producer offset `j` by acceptor from the same DC.
2. Acceptor performs sequentially:
	1. Store and flush element the element `e_i,j`
	2. Perform in parallel:
		- Broadcast `e_i,j'f` to the other acceptors.
		- Send `i,j'f` to proposer.
4. On receving `i,j'f` by proposer initiate consensus:
	1. Phase I: broadcast other proposers the map of `{i,j}_d` for the all DC where `{i,j}` is a map of `i,j`.
	2. Phase II: on receiving `{i,j}_d` from different DC marked by `d` do the following:
		- combine all `{i,j}_d` for all `d` from DC including itself, the combination is the minimum of all `j` for specific producer `i` for all `d`, the result is just `{i,j}`;
		- flush the result `{i,j}` only if the result is the same as received from other DC nodes meaning that others received `e_{i,j}` elements;
		- broadcast the result `{i,j}`;
	3. Phase III: on receiving flushed the same data from all nodes: commit the meta.

Delay:

0 -> Phase I: +0.5 RTT (send to another acceptor) +flush (before sending to proposer) = 0.5 RTT 1 flush
Phase I -> Phase II: +0.5 RTT = 1 RTT + 1 flush
Phase II -> Phase III: +0.5 RTT +flush = 1.5 RTT + 2 flush

### Basis invariant

2 commit the data the following invariants must be preserved:

1. Nodes cannot commit another sequence: _uniform_ property.
2. Nodes cannot forget the committed sequence: _unforgiven_ property.

_Nodes cannot commit another sequence_: is ensured by: all nodes have the same map `{i,j}` in memory.
_Nodes cannot forget the committed sequence_: majority of the nodes has flushed data and metadata on the Phase II completion.

Thus __Phase II completion__ is critical (both must be valid):

1. All nodes have the same data in memory => _uniform_ property.
2. Majority nodes have persisted that data including metadata => _unforgiven_ property.

## Store Memory Saving

Let's consider the following:

1. 3 nodes.
2. Data is split for 3 shards.
3. Each shard has 3 the same nodes: but first shard excludes the first data node etc.

Thus 3 shards `s1`, `s2` and `s3` with 2 replicas `r1` and `r2` and metadata replica `m` might have the following deployment:

Node 1: `s1.m  s2.r1 s3.r1`
Node 2: `s1.r1 s2.m  s3.r2`
Node 3: `s1.r2 s2.r2 s3.m`

Capacity is increased from 33% to 50% (factor 1.5).

The same procedure could be applied for 5 nodes etc.

## RTT Optimization

In normal case timeoutless algorithm allows to commit the data in 1.5 RTT (e.g. for 5 nodes). Whereas the simple calm algorithm allows to commit in 1 RTT. Thus it would be a good idea to commit the sequence in 1 RTT under good conditions instead of 1.5 RTT as normal algorithm provides.

### Group Assigning

The idea is based on the group assigning in advance. Let's consider 5 nodes. By obtaining messages from 4 nodes we could create the group event if we don't have replies from other nodes with the same sequence of messages. The thing is that 4 nodes preserves majority because in case of failure of 2 nodes we still have 2 nodes that is more than 1 another node (the 5th node).

Thus the group must preserve invariant: `assigning_group - max_fail_nodes > nodes - assigning_group`.

If `nodes` is odd thus `max_fail_nodes = (nodes - 1) / 2` => `assigning_group > (nodes + max_fail_nodes) / 2 = (3 * nodes - 1) / 4`

If `nodes` is even thus `max_fail_nodes = (nodes - 2) / 2` => `assigning_group > (nodes + max_fail_nodes) / 2 = (3 * nodes - 2) / 4`

`nodes` | `min_assigning_group` | `delta`
2 | 2 | 0
3 | 3 | 0
4 | 3 | 1
5 | 4 | 1
6 | 5 | 1
7 | 6 | 1
8 | 6 | 2!
9 | 7 | 2!

Actually, the assigning group for 5 nodes can be up to `nodes` thus it could be 4 or 5 depending on the commit sequences from other nodes. The thing is that 1 RTT is obtained only in situation when all 4 nodes in assigning group have the same sequence and they don't interfere with each other. If there is a conflict the number of RTT is increased (sometimes event more). Thus the most reliable way is to use `assigning_group == nodes`.

### Algorithm modification

To implement this algorithm we need to preserve invariant that there can be group and failed nodes might commit within that group. Thus we need to sort the messages based on maximum of number of sequences instead of majority: if node have 2 the same sequences and 1 another sequence it should use 2 the same sequences while majority could be equal to 5.

### Group Change Heuristics

If the node failed for significant period of time it's better to remove it from the nodes group to preserve minimal commit RTT based on Group Assigning optimization.

The idea is simple: on any commit preserve number of committed groups involved in the sequence. If in window of that sequence there is a majority of participant that below the expected number we could choose to remove it for a period of time.

E.g. window is 10 commits. If 3 commits with 5 nodes and 7 commits with 4 nodes we could decide to shrink the group using 4 nodes.
