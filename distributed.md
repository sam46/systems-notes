
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
