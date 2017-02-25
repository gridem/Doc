# Replicated Queue

The document describes the product: replicated queue aka *replque*.

## Introduction

Current systems Kafka etc doesn't have CP property: under network split they may loose important data. This is due to architectural limitations:

1. Kafka is based on Zookeeper to elect the leader. To preserve consistency under network split it must check leadership on each incoming request.
2. RabbitMQ - based on Erlang and mnesia DB which is CA meaning that it loose C under P.

## Design

Design is very simple:

1. There is persistent queue.
2. Queue is replicated using *replob*.
3. GC periodically cleans outdated elements.

### Persistency

Persistency can be achieved by using one of the following approaches:

1. Use LevelDB, RocksDB, Sqlite etc to persist the actions.
2. Use WAL to store actions on a disk.

The second approach is preferable because we need to preserve high throughput that can be achieved by appending data only.

### Replication

Replob approach must be used to correctly replicate the data. It allows to achieve high throughput by using batching.

#### Replob Batching

On incoming request from client there are 2 possibilities:

1. There is no any requests on-the-fly. Thus we start incoming message immediately.
2. If there is on-the-fly request processing we add to the batch queue. On complete on-the-fly we close our batch queue and process this batch as a single message. On new incoming message we create new batch queue.

### Cleanup

We need periodically cleanup the very old entries. The approach is similar to circular buffer. There are several policies:

1. Time to live TTL.
2. Number of elements to preserve.
3. Limit the amount of memory to be used.

Several options may be combined.

## New Features

Here is a short list of new features:

1. Lua trigger. User may want to add specific behavior on add new element (and possible on subscribe/unsubsribe). E.g. send to another queue or perform some data processing:
    - stats collection
    - real-time map-reduce
2. Use the queue as a storage. E.g. to preserve just key/value.
3. Use as a lock. Clients may subscribe and wait for unlock operation.

Lua trigger is the best preferable option.
