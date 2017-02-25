# Distributed Ideas

## Sequence Id

The most important entity to provide consistent execution in distributed environment is using so called *sequence id*. The idea is to update *sequence id* on each write and check it on read operation. The basis was taken from the article [How to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html).

It allows to handle:

1. Distributed Locks.
2. Distributed Queues.
3. Exactly once semantics.
4. Avoid stale reads.
5. Fault tolerant execution.

Thus it's generic mechanism to avoid distributed inconsistencies between 2 interacting services.

## Distributed Transactions

The idea is to use set of tagged sequence id to take a snapshot of a view. E.g. after updating a wiki page I would like to search it. To be sure I need to get result only if after update page sequence id my index contain that page sequence id. That 'page sequence id' is unique for all pages and increments on any page updating. One can use the replque (replicated queue) to send and batch update and further waiting for process it.

## Replob Isolation

To avoid complexity on update it's convenient to split replob and replob usage:

1. *Replob core* allows to linearize operations.
2. *Service* cooperates with the *replob core* to apply its own actions.

### Replob Core Update

Core allows to linearize the operations from any clients regardless of the binary and update information. It inputs the message as blob and outputs the linearizable sequence of blobs in the same format.

The update procedure is the following:

1. Client executes update request.
2. Request goes through replob.
3. On commit: start new process with updated binary.
4. Optional for robustness. Check that new process started and replob is operated normally without crashes etc. To do that just invoke replob action and wait for that response. Verify payload: it must be the same as received on request.
5. Start switching procedure through replob.
6. On switching commit: transfer the rest of messages to new process for applying and committing. During that period work as proxy then stop processing after client has been switched on the new replob core instance.
7. Stop the process.

### Service Update

Service update is related to functionality when the group is growing. But there is additional complexity: the binary is changes thus you cannot use functions, only data.

The idea is based on snapshot transferring and switching:

1. Client executes the update request.
2. Request goes through replob core.
3. On commit:
    - take snapshot
    - start new process with updated binary
    - apply snapshot
4. Start switching procedure through replob.
5. On switching commit: transfer the rest of messages to new process for applying and committing. During that period work as proxy then stop processing after client has been switched on the new replob core instance.
6. Stop the process.

> **TODO**. There is a lag between snapshot applying and committing the first message. Need to reduce somehow that.
> **Note**. The idea is to switch binaries one by one based on the fact that client has platform independent messages e.g protobufs. Thus we switch internal messages replob on external messages (protobuf) replob.

Solution: split heavy object to smaller object and use update for those objects. Pros:

1. No additional memory penalty: only small delta is used and can be updated sequentially one-by-one.
2. Smooth update: the time can varies in a very different ranges: faster or slower, depending on split and parallel objects synchronization.

Requirement: need to map old functions to new functions.

### Functions Mapping

The idea is based on diff from local and remote. The function to be serialized is the following:

```cpp
struct X
{
    R method(A& a, B& b);
};
// converts to:
std::function<R(A&,B&)> fn;
// or:
std::function<R(const std::tuple<A,B>&)
```

> **TODO:** Return value is the question: how to return it and send back?

### Robustness

The idea is based on the following: instead of applying the unknown message on the service the service forks and executes the same binary and tries to apply the message on the forked binary. On successful operation it can apply internally. The only thing is the criteria of a *successful operation*.

It allows to isolate from the bugs like corrupted messages or invalid message that causes service to crash on apply.

## Joins

The idea is to create the special index on the fly that contains all necessary references. Those references requires only limited number of hops to read the data. Ideally, only created index contains links that contains leafs (not trees).

### Reduce

One of the possible solution to implement: use reduce function in map-reduce concept.

It can be implemented as follows. Scheduler creates *points of reduce* to store the streamed of reduced data there. On finalizing those points are closed and process it further.

### Indexing

Another idea is to use special index. Example:

`select t1.id t1.id_ext t2.name join on t1.id_ext=t2.id`

Can be achieved by streaming from `t1` to `t2` without any index: map on `t1` and result are going directly to `t2`.

Another example:

`select t1.id t1.type t2.name join on t1.type=t2.type`

Need to create index for `t2.type` to transform this case into the first case.

## United Execution

If the execution requires access to different replobs it's worth to split the whole system on the parts. Each part are operated independently by applying deterministic properties of the execution. It avoid addition synchronization for different unrelated objects. It's so called *ring of reliability*.

Thus execution must be deterministic.

## Consistent Hashing

Instead of doing consistent hashing the following approach could be applied. The idea is based on the fact that each table may have its own configuration. That configuration may spread the table across new replicas without significant reconfiguration penalty. It allows to significantly simplify the logic: instead of calculating the consistent hash you need to calculate just usual hash based on the specific set of nodes. The only thing is to preserve the set of nodes but it could be packed easily into a couple of bytes using packed bits where each bit represents the node index. Thus 3 nodes in 256 nodes set are represented as 3 bytes.

## DB Rehashing

The idea is based on the fact that doubling the nodes requires only half of data movement. Thus it's very efficient and simple because consistent hashing does the same but it requires more complex computations.

Usually factor of 2 is ok in many cases e.g. see `vector` implementations.

## Range Index

The simplest way to spread data uniformly across the replicas is to apply the hashing based on configuration of replicas. The only question is how to apply range based select statements.

The answer is to add elements on replicas based on hash and inside the replica use persistent storage layer that allows to execute queries with ranges e.g. RocksDB or LevelDB storage engines.

## Advanced Batching

Sometimes it's useful to prepare requests for batching. E.g. the task is to generate sequential `id`s. The operation may be described by the following class:

```cpp
struct IdGenerator
{
    Id nextId();
};
```

If we batch those commands they will have the same meaning and it will contain a lot of unnecessary bytes. Instead that we could convert this command into:

```cpp
struct IdGeneratorBatch
{
    // `RangeId` is a pair of begin and end `Id`
    RangeId nextId(int num);
};
```

Another idea is to concatenate the arguments into the list (or set) of arguments, e.g. for delete operation:

```cpp
void deleleRecord(Id id);
// will be converted to
void deleteRecords(set<Id> id);
```

So it looks like reduce operation where the key is the function and value is the arguments.

### Solution

The solution is to create adapter for the class and invoke it before going to consensus. Thus the sequence of operations will be:

1. Client executes method.
2. Client adapter packs the method and arguments and sends it to replica consensus.
3. Replica consensus accept the client message and immediately invokes it on adapter that creates batch.
4. That batch may use different policies for different classes and methods e.g. performs reduce operation on each method and combine the arguments using `list` or `set` or apply specific method.
5. That batch goes through the consensus.
6. After consensus it applies as usual and returns the results.

The only question is how to send reply for each clients. On advanced batching operations we should save the source to send reply later.

## Configuration Changes

Sometimes the cluster requires applying new configuration. That configuration must be spread across the whole cluster and each node must accept that configuration. The thing is that wrong configuration may broke the whole cluster event if it implements fault-tolerant strategies.

The idea to solve that issue is to apply the following technique. The configuration must be applied sequentially not immediately, part by part. If some part becomes unavailable for some time - it's ok to restore the state of that part. And the whole cluster will continue to operate normally.

Thus the receipt is: apply sequentially and wait for notification about health, then continue to apply for another part.

See [Testing Distributed Systems w/ Deterministic Simulation](https://youtu.be/4fFDFbi3toc) for details about possible failures and how to prevent them.

## Resource Allocation

It's important to control available resources like CPU or memory. One way to do that is to use the RAII approach:

```cpp
void perform_select_operation(const Operation& op)
{
    size_t items = calculate_affected_items(op);
    {
        // at the beginning of the block we allocate necessary amount of memory from resource manager
        MemoryResource mem(items);
        // constructor of `MemoryResource` ries allocate and blocks if there is no required amount of memory
        // ...

        // destructor releases the resource at the end of the block to allow another operations to proceed
    }
}
```

The same approach could be applied for CPU (cores), network or others.

But the most powerful idea is to not allocate resources in an abstract manner and in advance but using the real allocation mechanisms, disk etc. Like this:

```cpp
void perform_select_operation(const Operation& op)
{
    StartOperation operation("SELECT"); // constructor initializes resource checker and connects it to resource manager
    size_t items = calculate_affected_items(op);
    std::vector<operation_items> ops(items); // here the vector might be blocked by allocator that cooperates with memory resource manager
    // while destructor of vector deallocates memory and releases the resources
}
```

`StartOperation` designates that it's necessary to control the resource utilization (coroutines, disk access, network IO etc) and block if there is no required memory.

> The idea is based on scheduler of remote operations and control them.

## Allowed Reliable Distributed Executions

There are 2 types of executions:

1. *Persistent*: all mutations are performed through the same deterministic states and objects on all nodes.
2. *Transient*: all mutations are performed by single node that responsible to propagate the results.

> **Note**: Transient mutations can be performed by several nodes to validate the processing (see patent "Calculation Robustness" section).

### Allowed Persistent Mutations

Persistent mutations are characterized by a parallel execution by all nodes inside the group. Thus those mutation must be deterministic:

#### Allowed

1. Single threaded execution without any synchronization.
2. Synchronization where the object is the only source of mutation.
3. Pipelining where each pipe has the only source. The number of consumers may be more than one.
4. Single point of operations. E.g. replob is the single point and it allows to correctly implement exactly once semantics.

#### Forbidden

1. Concurrency are forbidden. E.g. using synchronization like mutexes, putting to channels with other producers etc.
2. Network communications with services including GET and PUT operations (const and mutation operations).

### Allowed Transient Mutations

It looks like the same as for persistent mutations.
