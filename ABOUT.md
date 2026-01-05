# Consular Raft

Consular Raft is an increment to Raft consensus to provide coordinated
scaling over multiple Raft groups.

For large data sets, i.e. state machines with large states, it is
desirable to partition consensus into multiple Raft groups. This
results in smaller and more manageable Raft logs and snapshots, and
greater concurrency if writes are spread out across data partitions
and consensus can be processed concurrently across Raft groups.

The approach here is that all the Raft groups must have the same
physical server as leader. This enables the leader server to process
all write transactions without requiring distributed transactions, and
to serve consistent reads without requiring data from other
servers. The motivation is to facilitate a broad set of workloads that
includes complex transactions and analytics.

NOTE: Cockroach Labs, Yugabyte, and others have also scaled Raft. The
approach here is different in that we require all the Raft groups to
have the same server as leader. Hence we could not reuse the scaling
work of others.

# Approach

We want to minimize changes to Raft. We designate a single Raft group
as global leader candidates, and all other Raft groups as workers that
actually maintain state.

The Raft group with the global leader candidates is called the
delegate group, and its members delegates. The leader of the delegate
group is the global leader and is called the consul. The delegate
group is a standard Raft group, without modification.

The worker groups are modified so that each member holds a reference
to its local delegate, which must be on the same physical server. The
leader of each worker group is always the member whose delegate is
consul.

Elections are held only within the delegate group. If a worker
requires an election, it asks its delegate to conduct an
election. Heartbeats are required only within the delegate group. When
a worker needs to check for a heartbeat, it checks if its delegate
received a heartbeat. The delegate group is a control plane, and the
workers a data plane.

This model works best when a delegate and its corresponding workers
all live in a single process. This has at least two benefits: (1)
accessing a delegate from a worker is a simple in-process dereference,
and (2) the availability and liveness of a delegate and its
corresponding workers are naturally coupled, because they are tied to
the availability and liveness of the same process.

# Read scaling

We want to gain read scaling by serving reads from replica servers and
parceling out read workloads among all healthy and up-to-date servers.

We provide a few read consistency options.

### Leader strict consistency

Leader reads are always strictly consistent. If a write commits before
a read arrives __at the leader,__ the read will see the write.

To prevent a deposed leader from serving inconsistent reads, a newly
elected leader delays committing incoming writes for a fixed delay
after its election. This delay is long enough to ensure that the
deposed leader times out and steps down before writes are committed by
the newly elected leader. Hence, leader reads are always strictly
consistent.

__Note:__ Writes are always strictly consistent. Writes always require
a majority quorum, so writes cannot be committed by a deposed leader.

### Replica strict consistency

Strictly consistent reads can be served by replicas. If a write
commits before a read arrives __at a replica,__ the read will see the
write.

To ensure a strictly consistent read, a replica receives the read and
then consults the leader to ensure the replica has applied all writes
committed before the read arrived __at the replica.__ If not, the read
is redirected to the leader.

### Replica bounded consistency

Bound-consistent reads can be served by replicas. When a
bound-consistent read arrives __at a replica,__ the replica determines
if it (the replica) is up to date within a bound N, and if not, the
read is redirected to the leader.

The consistency bound N is defined in terms of heartbeats. A replica
is N-bound-consistent, or up to date within bound N, if: (1) the
replica has received a heartbeat within the heartbeat timeout, and (2)
the replica has applied the Nth previous heartbeat's commit index,
where N is a positive integer, with 1 for the most recent heartbeat's
commit index.

Bounded consistency trades off strict consistency for gains in both
latency and throughput. Replicas perform the bounds check without
consulting the leader, improving latency. And by definition, the
bounds check results in fewer leader redirections than the strict
check, improving throughput.

### Replica session consistency

Session consistent reads can be served by replicas. If a client reads
the effects of its own write, the read will see the write. For
session-oriented use cases, session consistency provides strict
session semantics while enabling higher read performance and
availability than strict consistency.

Session consistency can be defined in terms of bounded consistency,
inheriting the latency and throughput benefits. The client can track
the commit index of its last write, and that commit index can be used
to set the consistency bound.
