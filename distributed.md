
# Clocks:
wallclock/ntp: bad, not always monotonically increasing.
OS's provide a monotonically increasing clock, tho it's supposedly buggy/unreliable.

### logical clocks:
more about order of events (i.e. timestamp is more like an index), and causality
#### implementations: 
lamport clock:
- single timestamp that gets increased as events occur in a (single) process, or as messages are sent/recieved between processes
- if (`a -> b` i.e. `a` [happened before](https://en.wikipedia.org/wiki/Happened-before) `b`, causally, not in actual wallclock time) then `LC(a) < LC(b)`
- but `LC(a) < LC(b)` doesn't imply `a -> b`: concurrent events `a || b` can still end up having `LC(a) < LC(b)` 
- so it gives a partial order: you can compare events, but not all events are (meaningfuly) comparable

vector clock: 
- gives each event a vector, e.g. `(0,4,1)`, where a coordinate represents events progression of a single process
- gives partial order (like lamport clock): `VC(a) = (1,2,3) < VC(b) = (6,7,8)`
- but it's clearer than lamport clock: concurrent events are obviously incomparable, e.g. `(0,0,1) vs. (1,0,0)`
- `VC(a) < VC(b) <=> a -> b` (unlike lamport clock, which is only 1 way)


# CAP
In the presence of a network partition (i.e. part of the system can't talk to the other part), we can only be:
- available (to clients) but in-consistent (the two partitions' states are not identitcal)
- un-available (to clients) but consistent (the two partitions have identitical state)


#### split-brain
inconsistency during a network partition, in a model that's supposed to guarntee consistenty

# Approaches to replication
- active: client sends to every replica
- passive: client talks to leader. leader handles replication
- lazy: client sends to any node, replication happens in the background (e.g. gossip protocols)

# Raft replication (etcd)
strong consistency: in the presence of a partition, some nodes/replicas are unavailable
consensus/majority/quorum protocol:
- always choose odd number of replicas/nodes
- clients talk to leader
- leader "commits" and responds to client **_only after_** ensuring majority of nodes recieved the request too (i.e. the leader does the replication)
- periodic re-elections. Candidate needs a majority vote to become leader
- every node maintains a log + other metadata (also written to disk for crash recovery)
- log contains the (application) state entries (+ election term with each entry)
- split-brain fault-tolerance during a partition:
  - minority partition's nodes can't successfully elect a leader, so they've all effectively become un-available, and they wont recieve any updates.
  - majority partition's nodes can elect a leader (if necessary), and just operate as normal
- log is used for getting crashed or unavailable/cut-off nodes back up to speed 
- log contains entire application state from beginning of time: 
  - gets big so _"Snapshotting"_ is employed: `[x=1,x=2,x=3,y=9]` can be compacted to `[x=1,y=9]`
