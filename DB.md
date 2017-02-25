# DB Ideas

[TOC]

## Multiindex

I can use multiindex container. Insert operation are persistent while get operations can be arbitrary. Thus I can change multiindex container (e.g. add new indexes) without breaking the structure.

## GC

GC can be applied for part of container. Just use the hash for key to split different keys uniformally.

## Database Index Access

Index can be accessed in different process without requiring data transferring between master process and clients.

The idea is to use MMS-like approach: mmap index file and use the underlying data as it is (use fixed address or relative addresses). Mmapping allows to concurrently use committed DB index in different processes without interaction with master process.

## Scalable Transactions

Hierarchy allows to scale the transactions. Let:

P0 is the value for the key
P1 is the table
P2 is the set of tables

P0-P2 is hierarchy. When we need to store the key we may use just P0 level of transactions, we don't need to linearize access for higher levels.

For the table operations we need to linearize our actions on P1 level.

For operations across tables we need to linearize our actions at P2 level.

### Dependency Manager

Linearization usually costs a lot. But we can parallelize our independent transactions e.g. transactions for different keys or tables. Thus we need to have lock manager (see very lightweight locking VLL article).

### Why is it Scalable

Because if we need to get a key from a table we just go to that table without interaction with transactions at P1 and P2 levels.

Independence allows to scale and higher levels.

### Consistency

Consistency is serializability. Let's consider the following example:

Registers: `a` and `b`

Transactions:

T1: a=a-1; b=b+1; (money transfer, amount=1)
T2: read a;
T3: read b;

Real sequence could be:
1. start T2
2. start T1
3. T1: write a->0
4. T2: read a->0
5. finish T2
6. start T3
5. T3: read b->0
6. finish T1
6. finish T2

Physical sequence: T2, T1, T3, and T2+T3 in single thread. In sequencial (and linearizable) consistency level T2 cannot be later then T3. Whereas serializable consistency may yield: T3, T1, T2.

### Consensus

All transaction managers must have consensus to reliably commit the changes. There is still the possibility:

1. Persist queue for actions on upper layers.

Thus we need to have a consensus only on the started transaction manager. It responsible to propagate other actions reliably (optimization option).

## Different DB Backends

The idea is to use different backends and make them fault-tolerant (see Patents: SQL Database Replication).

Start from LevelDB:

1. Create adapter around external interface.
2. Use it to linearize actions.
3. Need to store deltas: can be used another different storage engine.

It allows another users to utilize those ideas and adapt MySQL/SQLite and others => community efforts.

### Concurrency

After consensus to apply actions on different backend we can parallelize the access to DB by using:

- key hashing
- value: channel with coroutine that listens and applies the actions.

I need to play with the number of values inside that table to increase the concurrency.

#### Consensus Concurrency

Actually, I could apply consensus concurrent access to reduce operation latency. But it looks like increase CPU timings during handling.

## Hierarchy: Jumping

Consider:

- P0 is the value for the key
- P1 is the table
- P2 is the set of tables

If client requests 2 values for different tables => it should go to P2. But P2 can go directly to P0 if values are not interfere with P1.

## Big Hash Map

To store in-memory hash table it's reasonable to split it using fixes-size vector of elements:

```cpp
using BigHashTable = vector<unordered_map<string, string>>;
```

The idea is that vector uses part of calculated hash to calculate index. It looks like a tree with constant depth `== 2`.

### Vector Size

Let's assume that we have 10 MB buffer. Thus we could take a snapshot of all table by using `size(vector) * sizeof(hash_table_version)` where `sizeof(hash_table_version) == 8` for `uint64_t`. Total size is 8MB message.

Another consideration about size of each hash map:

    Let `sizeof(key) + sizeof(value)` is 1 KB = 1e3 B
    Let the RAM memory is 100 GB = 1e11 B
    Number of keys: 1e11 / 1e3 = 1e8
    Thus `sizeof(unordered_map)` ~= 1e8/1e6 ~= 100

    For persistent DB with size 10 TB = 1e13 B
    Thus `sizeof(unordered_map)` ~= 10000

### Discussion

Pros:

1. Independent persistency for each `unordered_map`.
2. Independent synchronization.
2. Natural parallel replob replication for each `unordered_map`.

Cons:

1. Minor complexity.

## Sharding Policies

There are 2 different techniques could be applied:

1. Hash based sharding, e.g. using consistent hashing.
2. Range based sharding: each replica has its own key range.

For range queries the implementation would be:

1. Hash-based: send request to all nodes and perform range query, then combine them.
2. Range-based: send request only nodes that contain necessary range.

The advantage of the first approach that it allows to parallel the execution.Drawback is also obvious: if there are a lot of shards it performs a lot of unnessary execution on remote nodes.

The best way is to have tunable parameter that mixes both approaches:

1. We split key space on ranges of keys.
2. Each range has hash-based additional sharding.

Thus we could control the parallelism by specifying number of shards for each range.

The interesting question for range-based sharding is: how to spread it accurately across the replicas?

## Absolutely Scalable Transactions

The topic describes the approaches to implement absolutely scalable systems and transactions for number of nodes more than 100'000.

### Sequencer

Sequencer is used to specify the sequence of operations by assigning the monotonic incremental unique identifiers. It works in the following way.

When number of nodes is less than specified limit => we could use single consensus node groups to assign the `id`. The algorithm is the following:

1. Batch requests from different clients for specified period or condition (e.g. next round).
2. Batch generates the range to be agreed on consensus single round phase. Range contains the number of id to be assigned.
3. Consensus returns the initial id for that range.

If the consensus cannot handle specified number of client then we do the following:

1. Split all clients unto client groups.
2. Each client group batch requests.
3. Single consensus node (sequencer) accepts batched requests from client and batches the batches.  

## Memory Tracking

One of the difficult part is the memory tracking and accounting. There are different policies:

1. Tracking per request.
2. Per client.
3. Per process.
4. Per cluster etc.

### C++ Memory Tracking

To implement this approach with C++ we need to use overriden `new` and `delete` function. `new` function may check the size to avoid accounting small objects. Size may be a configurable parameter.

To avoid allocation overhead we can use the following scheme: allocate all necessary memory region in advance using `mmap` without population and increase the size on demand. The allocation size may be the highest possible memory size for policies.

The problem here that we can allocation the memory for the whole process and next request may not have necessary memory. Thus we need to allocate by big chunks.

## Concurrent Transactions

The idea is to provide highly concurrent transactions.