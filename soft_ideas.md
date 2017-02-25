# Software Product Ideas

[TOC]

## Physics Emulation of Environment

Possible emulations:

1. Network.
  2. Latency
  3. Bandwidth
  4. Tube model with water
2. Disk.
  3. Hard disk
    4. Spinning time (7200RPM)
    5. Seek time
    6. Interface bandwidth, data bandwidth etc
    9. Durability
    10. `fsync`
  4. SSD
    5. Minimum blocks
    6. Random access
    7. Rewrite cells
    8. Lifetime
    9. Durability
    10. `fsync`
3. Processor: CPU cores
  4. Concurrency: need to calculate the schedulers
  5. Grains of tasks
4. Memory
  5. Allocations
  6. Fragmentation
  7. Per thread optimizations

### DB Emulation

To tune performance need to understand:
1. memory consumption
2. blocks to write
3. blocks to read
4. periods
5. latencies
6. SSD/HD parameters

### Distributed Algorithms Emulation (DAVE)

Network emulator with node failures. It can check all async paths for packets.

### Limitation

To avoid livelock we could introduce the limit on number of skipped round trips. E.g.:

1. client issues request at `timestamp == 3`
2. then it handles at `timestamp == 5`
3. thus the `timediff` is 2.

We could limit `timediff` for all requests at session. On receiving client issues requests for the next timestamp: `+= 1`.

## Real Emulator

I can create a network proxy to transfer a messages using failure rules:

1. Periodic disconnections (with possible message dropping).
2. Increase latencies.
3. Reduce bandwidth.
4. Network partitions.
5. Drop all inbound messages.
6. Drop all outbound messages.
7. Mixed scenarios.

> **Note**. Raft is not protected against dropping inbound messages. Consider scenario: heartbeat outbound packets from master are going well while inbound ones are not => cannot accept any packet.
> **Note 2**. Another Raft issue: it allows to have 2 leaders in case of split. As a consequence stale reads are possible. See [etcd investigations](https://aphyr.com/posts/316-call-me-maybe-etcd-and-consul) for more information.
> **Note 3**. Raft issue due to network bridging: `A<->B` && `B<->C`, but `A` cannot see `C` and `C` cannot see `A`. In that case if `A` is a leader then `C` timedout => it can be a leader. The situation stabilizes on `B` leader election.

## Distributed Scheduler

The idea is to have isolated process that allows to execute named command line. Name is important to avoid duplication on execution. Name must be unique.

The responsibility is to execute any command line with different policies like:

1. Retry on the nodes.
2. Execute only on the particular node.
3. Operation timeout. (Node: it can be wrapped by using `timeout` command).

### Chronos

In order to provide single clock execution one may generate special name for the cron-like distributed service, e.g. convert each time event to string and add unique name as a prefix.

Possible name is "DisEx" - distributed scheduler.
