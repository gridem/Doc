# Byzantine Consensus Algorithm

- mark message as invalid
- mark node as invalid
- sign the messages => can be validated and dropped, can be broadcasts
    - sign out messages => can proof the invalid sender
    - need to think about removing messages from invalid sender?

## Common Considerations

Basically, the common flow is the following, assuming that there is no failures, only byzantine failures:

1. On client message: broadcast.
2. On receiving broadcast with single message
    3. if no message => broadcast empty message
    4. if there is client message => wait
3. On receiving messages from all nodes:
    1. Merging based on majority.
    2. Broadcast merged data
4. On receiving messages from all nodes again:
    1. Merging based on majority.
    2. If the merged data corresponds to merged data from the previous step => commit

### Problems

- How to commit the data?
- How to merge the data?
- How to deal with invalid message from node? Remove it from list or remove message from list?
- What to do if the number of byzantine nodes are more than expected? How to preserve correctness (e.g. non-byzantine algorithm preserve safety even when number of failures more than expected)?
- And finally: what to do with mixed failures: byzantine + crash + network?

## Authenticated Byzantine

Authentication prevents from messages:
- dropping
- changing or replacing
- generating new one

Thus it can spread from another nodes and the destination allows to validate and ignore it if it founds them invalid.

The key here is to add generation number for each message. Signed messages contain:

1. `session_id` - current session identifier. User can reset sign generator by specifying another `step_id` and new `session_id` is generated. Another instance (client) must verify that the `session_id` is the same from the beginning of the consensus process.
2. `step_id` - corresponds to current replob step identifier and must be equal to this. Generator increases `step_id` on each new message incoming.
3. `node_id` - node identifier. User specify it on new session creation and it's not changed across the calls.

### API

```cpp
struct Signer
{
    SignSession create(node_id);
};

struct SignSession
{
    void setNodeId(step_id); // can be invoked only once and must be invoked before sign
    Signature sign(view);
};

struct Signature
{
    Buffer sign;
    session_id;
    node_id;
    step_id; // incremented on each sign call
};
```

Sign generator can be external service (can be affected by another byzantine problem) or physical device (preferable).

### Invariants

1. Receiving all signed messages. It's allowed to receive messages not from all nodes e.g. node 1 can broadcast messages for 1 and 2 node thus node 3 is allowed to obtain request from node 1 only and not to wait for node 2 message
2. On non-receiving message from the node it must obtain requests from all nodes with reduced set of nodes.

## Masterless Algorithm

We can tolerate `n = 2*b+1` where `b` is the number of Byzantine nodes `byzantine_nodes` and `n` is a total number of nodes `nodes`.

### Invariants

Because the minority of nodes may introduce any message (commit, remove from group etc) the only way to proceed is to use majority voting for each decision.

Commit requires that the number of nodes that agreed on the sequence to be committed will be the same on the number of nodes:

> `commit_group - byzantine_nodes >= majority`

If `nodes = 2*byzantine_nodes + 1` => `majority = byzantine_nodes + 1` => `commit_group >= majority + byzantine_nodes = nodes` => `commit_group = nodes`

But if we consider the example:

```
nodes = 5
byzantine_nodes = 1
majority = 3
=>
commit_group >= 4
```

And there is possible to use timeoutless algorithm. Otherwise it impossible.

Another interesting case:

```
nodes = 4
byzantine_nodes = 1
failed_nodes = 1
majority = 3
=>
commit_group >= 3
```

See that `byzantine_nodes + failed_nodes > nodes - majority` that is impossible for odd number of `nodes`! Thus 4 is the minimum preferable value to demonstrate the advantage of byzantine failures where single node may crash and additionally another single node may behave arbitrary (byzantine failure).

### Full Description

_First Phase_. Each node sends the proposal message. Proposal message must contain `step_id` and message to be committed - carry message. Proposal message must be signed and must not be changed.

Each node may generate only single carry message for each step otherwise the node is byzantine node. If there is no any carry message for that step the node must generate empty message and do the same actions as described above (add `step_id` and sign it).

_Second Phase_. On the second phase the node wait for responses from all nodes and create a set of carry messages for particular step. That set must not be changed even if the node (byzantine actually) decided to change the carry message for the same `step_id`.

On obtaining all carry messages from all nodes it fixes them and generate carry set to be committed. Carry set must be signed and broadcasted (actually, signing is not required because it contains a list of all signed messages (actually, it's not true and we MUST signed it because the message may contain different properties like 'commit', 'node excluded' etc and they must correspond to a carry set)).

_Third Phase_. Each node waits for carry set from all nodes. It may commit the message if it receives the same set from all nodes. In that case it adds the property 'commit' and include committed set (message must be signed).

The node accepts only messages that contains the full carry set (e.g. the carry set from all nodes). But there are several possibilities:

1. Carry set is not the same.
2. There is the node where no responses.
3. Mixed scenario.

#### 1. Carry set is not the same.

It means that byzantine node(s) generate different proposal message for different nodes. In that case we compare signed proposals from all nodes and add nodes that changed the proposal decision to the list to be removed as byzantine nodes. That list means that the node will be removed from the group and all messages will be discarded.

The node must receive the number of properties at least not from `majority` but from `byzantine_nodes + 1` because we need to ensure the correct behavior in case of `byzantine_nodes` byzantine nodes (`+1` would be enough).

#### 2. There is the node where no responses.

It means that we don't have carry set from the nodes due to lack of node. In that case we add the property to be excluded from the group and broadcast it. Number of messages to ensure the exclusion must be at least `byzantine_nodes + 1` as described in previous item.

#### 3. Mixed scenario.

In that case we add property with failed and byzantine node(s) as described before.

#### Termination

Termination occurs on commit phase. The message may be committed if the nodes send the `byzantine_nodes + 1` messages with commit property.

A markers to commit the message:

1. Received `majority + byzantine_nodes` the same full carry sets.
2. Received `byzantine_nodes + 1` with commit flag.
3. `byzantine_nodes + 1` acknowledged removed nodes with `removed_nodes + agreed_nodes = nodes` where `agreed_nodes` is the number of nodes that has the same carry set. `agreed_nodes >= majority` => `removed_nodes < majority`. `removed_nodes = byzantine_removed_nodes + crashed_removed_nodes`. Committed carry set must exclude proposals from `byzantine_removed_nodes`.

## Commit Invariants

For simplicity let's consider firstly non-byzantine faults of the nodes. And later generalize the approach.

### Non-byzantine Case

2 commit system:

1. Node group commit.
2. Sequence commit.

On node failure we must shrink the group firstly and then commit the sequence. Group shrinking means that we must commit the new group with excluded the failed node.

> __Node group commit invariant__: all nodes from the new group respond with that group or less. The size of the group must have majority: `new_group_nodes >= majority`. _The invariant_: node group must exclude all messages outside the group to preserve next invariant.

> __Sequence commit invariant__: majority of nodes `commit_group_nodes` must respond with the same sequence and that sequence must contain the messages from all nodes (possible empty messages) within the nodes group. _Sequence invariant_: sequence must contain all nodes responses to avoid additional message within the group. _Majority invariant_: the rest non-failed nodes may commit the same sequence anyway.

Those parts are evolved as independent strategies. _Node group_ changing allows to proceed with sequence commit in case of failed nodes preserving _liveness_. _Nodes group invariant_ together with _sequence invariant_ allows to avoid adding new messages that may break the _safety_. _Majority invariant_ allows to restore the committed sequence in case of failure preserving _safety_.

### Byzantine Case

In byzantine case the sequence commit invariant must be extended:

`commit_group_nodes - byzantine_nodes >= majority` (_majority invariant_). The reason is that byzantine nodes may lie about the sequence and the majority must provide the correct answer at any case.

TODO:

1. Less group in message from node group commit: node may commit the sequence while others are not due to discrepancies and majority failure.
2. Describe `byzantine_nodes = 1`, `failure_nodes = 1` and `nodes = 5`. And commit case where `byzantine_nodes + failure_nodes < majority`.
3. Several node group commits without sequence commit for the same step.
4. Transfer node group commit to the next step.
5. Interaction with next step in case of parallel commits.
