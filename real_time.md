# Real Time Ideas

## Possible Optimizations

Persistence - synchronization between nodes and saving the state on the disk
Node reassigning - reassigning the node on-the-fly by reseting current and starting new task without state transferring and synchronization.

> *Note*. Stateless may be still nondeterministic e.g. timer events or random number generator.

Table of allowed optimizations for different types of actions.

| Type | Asynchronous persistence | Persistence skipping | Node reassigning | Speculative execution | Synchronous Persistence |
|---|:-:|:-:|:-:|:-:|:-:|
| Stateless | + | + | + | + | + |
| Stateful deterministic | + | + | - | + | + |
| Stateful nondeterministic | - | + | - | - | + |

Operations:

1. Scatter - deterministic, due to original sequence and asynchronous sending with exactly once semantics
2. Gather - nondeterministic due to requirement of preserving the same sequence because it depends on the accept timings

Destination at any time may ask nonacknowledged entries. But once it acknowledge it cannot get again.

## Map Reduce

Real-time map reduce consists of channels with deterministic or nondeterministic behavior.

### Nondeterministic Channels

Let's consider nondeterministic behavior as the most complex part.

The most performant implementation is the following. We have 3 nodes:

1. Master node to perform calculations.
2. Hot standby node to save calculations results and switch to on crash.
3. Arbiter to control crashes.

Let's suppose that we have output that is nondeterministic. In simple case we must persist the data before sending to data consumer. Consumer at any time may request previous data and we must send the same data again and again.

Data persistence in case of failure requires applying consensus thus adding significant latency (RTT + flush time). To avoid that latency in case of no failure we could use the following approach.

We send the data to the consumer without flushing. Current data is *unack* meaning that there is no acknowledgement yet. On distributed and persist data we *ack* the previous sent data.

In case of failure we allow to send another *unack* data with the same offset again. But we preserve the invariant that *ack* data cannot be changed.

If consumer receives the new data with old offset it drops the previous progress up to that offset and recalculate the data. The data cannot be *ack*ed by consumer until the calculation based on completely *ack*ed data and persisted and distributed as well.  
