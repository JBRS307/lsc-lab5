# CAP Lab Report

**NOTE**: Quorum in CP context means `floor(n/2) + 1` nodes.

## Exercise 1 - CP, one node isolated

Before partition: all nodes return the same value; increments are immediately visible on all nodes due to strong consistency.

After isolating node 3: node 3 becomes unavailable - both `get` and `add` operations time out on it (no quorum). Nodes 1 and 2 still form a majority (2 out of 3) and remain fully operational.

After healing: node 3 rejoins the CP group, the Raft leader replicates the missed entries to it, and all nodes converge to the same value.

## Exercise 2 - CP, all nodes isolated

After isolating all nodes from each other: no node can reach a majority (each has only 1 out of 3 votes), so every `get` and `add` operation times out on every node. The system sacrifices availability to preserve consistency.

After healing: nodes re-establish communication, Raft re-elects a leader, and all nodes return to the pre-partition state (no divergent writes occurred).

## Exercise 3 - AP, one node isolated (PNCounter)

After isolating node 3: all nodes remain available. Each node independently accepts `get` and `add` operations. Increments applied to one side are not visible on the other side during the partition.

After healing: each node's local increments and decrements are merged using the CRDT semantics - the final value equals the sum of all operations performed across all nodes, regardless of partition. No data is lost.

## Which data structures require Raft, and why?

CP systems require the Raft consensus algorithm. Raft ensures that all nodes agree on a single, ordered log of operations before any write is considered committed.
After a partition heals, Raft's role is to bring lagging nodes up to date: the current leader replicates all log entries the rejoining node missed, restoring a consistent state across the group.

## PNCounter session guarantees: RYW and Monotonic Reads

**Read-Your-Writes**: any read issued after a write in the same session will return a value that reflects that write (or a later one). For `PNCounter`, after a client increments the counter, its subsequent `get` will never return a stale value that predates its own write - even if the operation was served by a different replica.

**Monotonic Reads**: once a client observes a certain value, all subsequent reads in the same session will return the same or a greater value. A client will never "go back in time" and see an older state.

These are called **session guarantees** because they only apply within the scope of a single client session. Two different clients may observe values in different orders. The guarantee is not global consistency - it is a per-session contract that makes the system feel coherent to each individual client without requiring coordination between all replicas.
