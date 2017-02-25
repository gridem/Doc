# Streaming Processing

The article describes how to use streaming processing under different scenarios:

1. Transaction processing.
2. Replication and synchronization.

## Entities

The paragraph describes the useful fault-tolerant entities to build distributed and fast systems.

1. __Replob__: replicated object concept. See articles.
2. __Replact__: replicated action. Fault-tolerant handlers with sync/async persistency and speculation.
3. __Replque__: replicated queue. Fault-tolerant queue like channel with bounded/persistence properties and autobalancing.

### Speculation

Speculation is important property to decrease the latency in most scenarios. The idea is based that if we know that we have an agreement on sequence of operations/actions/data etc without properties like persistence and fault-tolerance (due to replication) we may still continue our operation and later to ack about those properties to commit it by receiver. It may dramatically decrease the latencies but should be accurately used between different entities. Each entity must be ready to accept intermediate results and later on commit them.

## Transactions

Client may use the following algorithm:

1. Open _replque_ to store data to the database.
2. Streaming the data to write data to the database.

Streaming looks like the transactions.

If we have different clients to write to the same database we must reuse the same _replque_.

If user wants to write to 2 different databases we may use single _replque_ and write data to those databases. Or we may create another destination _replques_ for databases and use replication approach with _replact_ (see below). Thus it looks like we create tree dynamically based on clients activities, optimizing the behavior.

It's good for streaming and bulk processing (like MR operations). But it's no so good for single shot operations that involves only several rows at once. But we may consider caching creation of _repl_ objects.

## Replication

The data could be replicated across different datacenters. The following scheme could be applied:

1. Client sends the data into _replque_.
2. _Replque_ connected to _replact_: _replact_ reads data from source _replque_ and distribute it to destination _replques_.
3. Destination _replque_ connected to _replact_ that puts the data into the table.

Benefits:

1. _Replques_ may have bounded queues to control the number of elements in the queue. E.g. one may consider that any replicas must have the difference not more than 1 hour. Thus we limit all destination _replques_ and it blocks _replact_ thus blocking clients to write the data.
2. Blocking clients may have another policy that connected to source bounded _replque_. E.g. one may consider to store the data at least for 1 day. Thus source and destination _replques_ may have different bounds depending on requirements.
3. _Replacts_ together with _replques_ may utilize the speculation approach to significantly improve the performance avoiding
unnecessary delay to consistently replicate the data using consensus.

The sequence of handlers and queues are known as _pull-push approach_.

## Generic Speculation

One of the possible way to significantly decrease the latency is to use speculation. The speculation can be used e.g. between _replact_ and _replob_ or between _replact_ and _replque_. _replob_-_replob_ interaction is under investigation (due to the fact that _replact_ is a single source of operations while _replob_ is invoked on all replicas).

The idea about generic speculation is based on the fact that the _replact_ should wait for state replication based on _replob_. It continues operation and sends to next _replact_ but with marks that this message still need to be confirmed. Next _replact_ may replicated it's state but with mark that the input and output is not confirmed yet. When the original _replact_ sends the confirmation mark the next _replact_ may send confirmation mark immediately. The reason is that the confirmation mark can be recalculated in case of crash of previous node(s)/_replact(s)_. Thus the confirmation mark may be replicated asynchronous.

### Failover

_Replact_ failover should include the `generation_id` that must be sequentially increased if the reassigns to the new node. The receiver must check for the highest `generation_id` number and must drop any other requests with lower values of `generation_id`.

### Replact and Timeoutless Integration

One important question is how to preserve the liveness of _replact_ without significant waitings and timeouts.

The best solution is the following. We should use timeoutless algorithm. But the algorithm itself doesn't preserve the liveness of the node and _replact_ as well. Thus we need to:

1. Periodically ping the nodes. Ping command must be the same meaning that pings from any nodes is equal to each other. The equality provides faster consensus convergence.
2. Check the list of nodes involved in the recent pings. If those lists don't have some nodes for a some number of steps (e.g. 5 or 10) it means that node has some issues and _replact_ may be reassigned to another node with increasing `generation_id` number (see failover section above). The failover lag in that case is `~= number_of_steps * RTT`.

### Replact Interactions

The base idea is to use interaction between _replacts_. The thing is that _replobs_ is not capable to send message and control message receiving and handling incoming messages. It's responsibility for the single actor - _replact_.

To preserve fast speed and fast failover the following methods could be applied.

#### Hot-standby

The most performant case is to use hot-standby approach. _Replact_ sends it's state totally asynchronous including consensus with disk persistence. Another replica just repeats the committed but not-yet-flushed state on disk. The reason of not waiting for flush is that it's important for majority nodes failure case only. In that case availability goes off and it's not necessary to preserve any state.

Confirmation mark must be sent right after disk flush to preserve unforgotten invariant.

#### Partial Commitment

Partial commitment allows to easily recover from abnormal speculation and saves more memory avoiding another replica to execute the same sequences of operations.

The price is that on failover another replica must recreate _replact_ from the last known state that may require some time. Also it increases the latency by at least 1 RTT.

#### Semi-partial Commitment

To preserve crashing one node for the price of 0.5 RTT we could do the following. We have 2 distinctive nodes for single _replact_: the first node accepts the messages from other _replacts_ and sends it immediately to the second node. Second node contains _replact_ and proceeds the actions. Thus in case of failure of node with _replact_ we may still preserve the actions and we don't need to rollback the operations.

#### Client Commitment

Instead of sending from client -> 1st -> 2nd node the client may send request to the 1st and 2nd node in parallel. Thus we don't need to spend 0.5 RTT time.

### Speculative Execution

In normal situation _replact_ is executed using single node. Actually to save the speed and state on failure one may use the scheme of using parallel actions execution using `>=2` nodes.

## Replact Types

Depending on deterministics the following _replact_ types could be introduced:

1. Fully deterministic. Both actions and results are deterministic and doesn't depend on any circumstances.
2. Logic deterministic. Only logic is deterministic but it may produce different results under different execution.
3. Fully nondeterministic. Both logic and result may be nondeterministic meaning that we can use another actions under next execution.
