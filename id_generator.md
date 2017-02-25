# Id Generator

## Problem Statement

Need to generate unique ID's and preserve invariant: if someone has seen the ID, generated new ID must be more than seen.

## Algorithm

### Incorrect

Incorrect algorithm is to preserve single master node and allocate ID in advance through consensus. The thing is that it's impossible to have single master/leader and to guarantee it at any time.

### Correct and Fast Algorithm

We need to use 2 consensus algorithms: in-memory and persistent. In-memory consensus works only in memory and doesn't store any data on the disk. While the persistent consensus is responsible to store the data on the disk at any commit.

Algorithm:

1. In-memory consensus allocates new range of id's from persistent consensus.
2. Persistent consensus commits the latest allocated id.
3. Client requests for the id from in-memory consensus.
4. In-memory commits the requested ids and respond to the client. If there is no available id => goto 1.
5. On in-memory consensus failure (majority of nodes) it must goto 1.

Analysis:

    client -> in-memory(batch, commit) -> client

    commit = 1RTT
    batch = 1RTT (0.99%)
    client <-> = 1RTT

    total: 3RTT

    In-memory -> persistent

    In advance allocating => 0RTT

Persistent batching only analysis for incorrect algorithm:

    client(batch) -> persistent(commit)

    commit in advance allocating => 0RTT
    client <-> = 1RTT
    batch = 1RTT (0.99%)

    total: 2RTT