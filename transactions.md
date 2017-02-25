# Transactions Processing

The document describes distributed transactions processing and different optimization techniques that could be applied.

## 2PC Hash Rebalancing

In order to deterministically randomize the writes to several nodes using 2PC we need to choose somehow the coordinator node. The obvious way is to randomize the node.

But it's better to do the following. We generate hash value based on e.g. key, sharding key and node (shard). For each row we calculate the hash and choose the minimum. That row with minimum hash value belongs to particular node and that node is going to be a coordinator.

The idea is the following:

1. If we have 2 sets for 2 nodes and size of the 1st set is much less than 2nd thus the most probable coordinator node is taken from 2nd.
2. If transactions have the same write sets with n nodes thus the sequence of application will starts from the same coordinator slightly avoiding conflicts.
3. The same consideration could be applied for partial equality of the sets from different transactions.

## 2PC across DC

Traditional RTT timings for 2PC between 2 datacenters:

1. Client sends the write sets to coordinator: +0.5
2. Coordinator sends first phase to another node to prepare: +0.5
3. Node sends back ack: +0.5
4. Coordinator sends 2nd phase to commit: +0.5
5. Node sends back ack: +0.5
6. Coordinator sends back to client OK: +0.5

Totally: 3 RTT DC (across DC).

Actually, 4 and 5 could be avoided because in that situation transaction will be succeeded anyway. Thus we have 2 RTT DC.

Optimized solution:

1. Client sends the write sets to set of nodes: +0.5
2. Set of nodes broadcasts ack to set of nodes: +0.5
3. Set of nodes sends back to client: +0.5

Totally: 1.5 RTT DC.

Actually, 2 could be avoided:

1. Client sends the write sets to set of nodes: +0.5
2. Set of nodes broadcasts ack to set of nodes and client (!!!): +0.5

Totally: 1 RTT DC. This is theoretical minimum.

### Hierarchical Sending

Sometimes it's very costy to broadcast all the data across all DC. In that case for group of nodes from a set of nodes we may choose a single one to represent the group of nodes. If we assume that the group of nodes is equal to whole set of nodes that transaction has thus it reduces to the case with single coordinator (normal 2PC commit). But the most generic case is the trees of nodes.

> __Note__. Client must send to the fault tolerant proxy that will represent the client wishes. The thing is that when the client fails while sending the writes to the nodes the transaction may stuck.

## Distributed Snapshot

Each _replob_ transactional rows state may have the following states:

1. _In progress_. Row currently is in progress using 2PC and potentially could be aborted.
2. _Committed unacknowledged_. Row has been committed successfully but the knowledge of commitment is not spread across the nodes involved in that commit. That row has a list of participant nodes.
3. _Committed acknowledged_. Row has been committed successfully with acknowledgement from the participants.

Snapshot cannot contain _in progress_ rows because of aborting. _Committed acknowledged_ must be present because every participant knows and committed rows as well. The only question about including the _committed unacknowledged_ rows.

The thing is that _committed unacknowledged_ rows are actually committed meaning that 2PC proceed successfully and participants eventually committed the entries. Thus we need to include those rows into snapshot but we need to cooperate with unacknowledged participants to ensure that their snapshot contains those nodes because they may be still _in progress_ state.

Thus the protocol is the following:
1. Client broadcasts the intention of distributed snapshot for the nodes.
2. On receiving snapshot request: if there is no _committed unacknowledged_ and _in progress_ => take snapshot immediately.
3. If there is _committed unacknowledged_ or _in progress_ with participants that in broadcast request: send _snapshot ack_ with list of _committed unacknowledged_.
4. On receiving _snapshot ack_ from all participants merge the list and update _committed acknowledged_ based on merged list. Snapshot must be taken based on merged list.

### Snapshot Speculation

In order to increase the performance of taking distributed snapshot the following algorithm may be applied:

1. On receiving snapshot request from the client take _speculative snapshot_ based on _committed acknowledged_ and _committed unacknowledged_. The thing is that merged list will definitely contain those lists and may be some rows from _in progress_ list.
2. The node may start processing records using that _speculative snapshot_.
3. Further on receiving and processing _snapshot ack_ obtaining the merged list we could postprocess them and include to results: join with results based on _speculative snapshot_.

> Dealing with delete operations in _in progress_ list. The thing is that delete operation may have some complication processing. Thus it's better to do the following: include deleted rows from _in progress_ list into _speculative snapshot_ and treat those rows as added if _snapshot ack_ merged list will not contain _in progress_ rows (like the holes introduction in pnp-transitions).

## 2PC and Deterministic Transactions

2PC transactions are treated opposite in comparison with deterministic transactions. The thing is that 2PC may take place across different entities in any order while deterministic transactions have global ordering to schedule them deterministically meaning that we could reproduce the result in case of failure based on the sequence of transactions to be replayed.

The drawback of 2PC is obvious: we need to resolve somehow deadlock or livelock behavior. Deadlock occurs in case of waiting for locked rows to be unlocked, livelock occurs in case of aborting transaction when we have locked rows belonging to another transaction.

To resolve 2PC issue one may consider using deterministic scheme to arrange transactions avoiding deadlock on conflicts. When we know the correct order we may proceed those transactions step-by-step without deadlocking.

But 2PC has significant overhead. Together with global ordering requirement it looks like the worst mix from two worlds.

Actually it's not when we consider speculative deterministic 2PC commit. The idea is the following:

1. We start our transaction as 2PC.
2. In parallel we try to assign global order for our transactions based on timestamp generator or special consensus algorithms to generate `id`s.
3. If there is no any conflicts => commit without waiting for timestamp generator.
4. If there is conflicts we need to wait until we have global ordering.

Once we obtain the order we could proceed with committing our transactions based on VLL (very lightweight locking) mechanism.

### Distributed Lock Table

Table locks may have different layer/nodes to process. The idea is that we lock the rows and wait them.

If there is no any conflicts on any distributed lock table for particular transaction thus we might proceed it. If there is conflict we need to wait until we become the first in the wait list (see VLL). To become first we need to have the sequence number based on global timestamp. Thus on lock conflict we have to wait for global timestamp assignment. But it could be done in parallel (see speculative 2PC and deterministic transactions section).

### Possible Optimizations

One possible optimization is the following. In some cases deterministic transaction require avoiding waiting for writes when we have successful read locks. In that case we don't need for timestamp generator and writes to handle reads.
