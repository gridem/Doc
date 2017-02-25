# Queues

The document describes the fastest consistent FIFO queues.

## Goal

The goal is to implement consistent FIFO queue(s) using geodistributed datacenters with different writing/flushing/transmitting policies. Architecture must satisfy the following requirements:

1. Must utilize the overall bandwidth between DC.
2. Parallel disk writing.
3. Several independent queues with FIFO within each queue. Overall FIFO is optional.
4. Different policies for the queues.
5. Different failover strategies.
6. The smallest possible latencies under normal conditions.
7. Possibility to save client data under temporary unavailability of another DC(s).

## Solution

### Streaming Consensus Algorithm

1. Initial step: collect all offsets from all queues as map: `{queue_id, offset_data}`
2. Broad offsets.
3. Receive all offsets from all nodes.
4. For each `queue_id` start triggers:
    - generate deterministically final offset based on policy and offset_data from all nodes.
5. Broadcast decision.
6. Accept the same responses.
7. Commit data and start persistent process to save:
    1. Metadata `{queue_id, offset}`.
8. For each `queue_id` wait for triggers:
    - generate execution decision based on metadata persistent policy and other offset data policy
9. Reply back to the client about persistence and consistent commit.

### Request Policies

1. Wait for accepting (completely unreliable)
2. Wait for committing (not completely reliable).
3. Wait for persisting (completely reliable).

### Queue Policies

Client may specify 2 types of policies:
1. Precommit policy.
2. Postcommit policy.

The difference is the following:

1. Precommit policy increases the latency of operation.
2. Failing on postcommit policy stopping the all queues because it must support overall FIFO guarantee.

Thus there is tradeoff between precommit (to avoid unnecessary waiting on disk failure, write failure, unavailability, crash recovery etc) and latency increasing: *moving from post- to pre- increases the latency (cons) and increases the reliability (pros)*

## Torrent Behavior

When `offset`s of the particular queue lies too far the node may request another node to upload the data from the corresponding point. It could be done by source node or by destination node.

## Policies

### Per Nodes Policies

1. Recieved data from the network.
2. Data send to disk backend.
3. Disk backend accepted data and replicate it with factor `N`.

### Group Nodes Policies

All per nodes policies may be combined with number of allowed nodes:

1. Single node.
2. Quorum nodes.
3. All nodes.
4. Mixed behavior: per-nodes policies combined with different group policies.

### Example

Precommit rule:

    network_replication == N || network_replication >= quorum && disk_replication + network_replication > N

Postcommit rule:

    network_replication == N && disk_replication >= quorum

## Geo-distributed Low Latency Streaming Adaptive Timeoutless Downtimeless Masterless Consensus

1. **Geo-distribued**. The algorithm perfectly works between different DCs and is capable to correctly resolve different delays/issues between them without loosing consistency.
1. **Low Latency**. Provides low latency between producers and consumers.
2. **Streaming**. It can be used in separated asynchronous large data upload/download with sofisticated algorithms like torrent and synchronize access for the data.
3. **Adaptive**. It may adapt runtime characterisitics to avoid unnecessary delays by tuning the paramenters to be committed based on recent history from the nodes and disk/network latencies.
4. **Timeoutless**. Doesn't wait for node(s) responses to commmit the data.
5. **Downtimeless**. See list below.
5. **Masterless**. New outstanding technique without master.

Additional algorithm options related to downtimeless:

1. Hot node switching.
2. Hot software update.
3. Membership changes.

## Consensus Algorithm

### Safety Properties

1. The committed list of elements contains only proposed elements.
2. Each committed list of elements will be as a prefix of any commit on any node when committed size not less than the prefix.
3. Each proposed element will be committed only once meaning that the commit sequence will contain the only that element.

### Liveness Properties

1. Under good conditions if majority nodes are alive the proposed element from that majority will be eventually committed.

### Requirements

Each element must be sortable. It can be implemented by using several techniques:

1. Each element contains GUID.
2. Node proposes the element with a pair of `{SeqId, NodeId}`, where `SeqId` is monotonic counter for particular node `NodeId`.

### Description

Each _Proposer_ `i` accepts the element `e_i` on step `s`. Need to agreed on sequence of `e_i`.

1. Receive element `e_i` from _Acceptor_.
2. Broadcast `[e_i]`.
3. On receiving sequence of `[e...]`:
  - add new elements to the list of elements `l`
  - replace list of elements of receiver _Proposer_ with obtained one if the version is newer than current
  - add new elements to the list of current _Proposer_
  - check the majority of element. If there is a majority:
    - generate majority group based on the _Proposers_ in that group. Vote for that group.
  - sort elements that don't belong to majority group for current _Proposer_. Increase version.
4. Broadcast the voting list and current _Proposer_ elements list.
5. On receiving votes where `majority group == votes` => commit.

### Queue Extension

On commit list of elements just merge it by using `min`-based approach as before.

### Fsyncing

On step 4 `fsync` current state and broadcast the data with `flush_flag = true`.
On step 5 commit only when `flush_flag = true` for all nodes within group.

```cpp
using Elements = vector<Element>;
using NodesSet = set<NodeId>;

struct NodeInfo {
    int version;
    Elements elements;
};

struct Group {
    NodesSet nodes;
    NodesSet votes;
    NodesSet flushes;
    Elements elements;
};

using NodesInfo = map<NodeId, NodeInfo>;
using Groups = vector<Group>;

NodeId currentNode();
int majority();

struct Proposer {
    struct State {
        NodesInfo nodesInfo;
        Groups groups;

        bool operator==(State) const;

        Elements getElements()
        {
            return nodesInfo.flatten(_.second.elements);
        }

        void mergeElements(Elements elements)
        {
            auto&& info = nodesInfo[currentNode()];
            auto newElements = elements - info.elements;
            if (newElements.empty()) {
                return;
            }
            info.elements.append(newElements);
            ++info.version;
        }

        void mergeInfo(NodesInfo info)
        {
            nodesInfo.foreach
            {
                if (info[_.first].second.version > _.second.version) {
                    _ = info[_.first];
                }
            };
        }

        Groups generateGroups()
        {
            Groups result;
            bool sorted = false;
            for (i = 0..) {
                // countForEach returns list of {count, nodes} for each element at index i
                // .max() returns count and nodes that contains max count
                auto maxCount = nodesInfo.countForEach(_.second.elements[i]).max();
                if (maxCount.count() < majority()) {
                    if (sorted)
                        break;
                    sorted = true;
                    sort(nodesInfo[currentNode()].elements[i..]);
                    --i; // will restart for the same index i
                    continue;
                }
                NodesSet nodes = maxCount->first;
                result.append(Group{
                    nodes = nodes,
                    votes = { currentNode() },
                    flushes = {},
                    elements = maxCount->second.elements[0..i] });
            }
            return result;
        }

        void mergeGroups(Groups grs)
        {
            // TODO: merge smaller groups more accurate
            for (auto&& g : groups) {
                grs.find_if(_.elements == g.elements && _.nodes == g.nodes)
                {
                    g.votes |= grs.votes;
                    g.flushes |= grs.flushes;
                }
            }
        }

        void mergeState(State state)
        {
            mergeElements(state.getElements());
            mergeInfo(state.nodesInfo);
            auto newGroups = generateGroups();
            groups.swap(newGroups);
            mergeGroups(newGroups);
            mergeGroups(state.groups);
        }

        Elements toCommit()
        {
            groups.foreach_backward
            {
                if (_.votes.size() >= majority()) {
                    return _.elements;
                }
            }
            return {};
        }
    };

    State state;
    Elements committed;

    void propose(Element e)
    {
        State newState = state;
        Elements elements = { e };
        newState.mergeElements(elements);
        onBroadcast(newState);
    }

    void onBroadcast(State sourceState)
    {
        State newState = state;
        newState.mergeState(sourceState);
        if (newState == state) {
            return;
        }
        state = newState;
        broadcast(state);
        auto commitElements = state.toCommit();
        if (commitElements != committed) {
            committed = commitElements;
            commit(committed);
        }
    }

    void broadcast(State sourceState);
    void commit(Elements elements);
};
```

### Comparison

|Algorithm|Worst-best cross-DC RTT (*)|Comment|
|---|---|---|
|Raft|2|Wait for majority|
|Masterless|1|Wait for all|
|Timeoutless|1.5 (1 RTT for 3 nodes)|Wait for majority|

> __(*)__ Worst-best case: best in context of network and process reliability (no disconnections, no delays, no drops, no crashes etc); worst in case of topology: consumer, producer and master are in different DCs

Separated data and metadata comparison:

|Algorithm|Worst-best cross-DC RTT (*)|Comment|
|---|---|---|
|Raft|2.5|Wait for majority|
|Masterless|1.5|Wait for all|
|Timeoutless|1.5|Wait for majority|
