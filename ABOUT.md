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

The Raft group with the global leader candidates is called the senate,
and its members senators. The leader of the senate group is the global
leader and is called the consul. The senate is a standard Raft group,
without modification.

The worker groups are modified so that each member holds a reference
to its local senator, which must be on the same physical server. The
leader of each worker group is always the member whose senator is
consul.

Elections are held only within the senate group. If a worker requires
an election, it asks its senator to conduct an election. Heartbeats
are required only within the senate group. When a worker needs to
check for a heartbeat, it checks if its senator received a
heartbeat. The senate is a control plane, and the workers a data
plane.

This model works best when a senator and its corresponding workers all
live in a single process. This has at least two benefits: (1)
accessing a senator from a worker is a simple in-process dereference,
and (2) the availability and liveness of a senator and its
corresponding workers are naturally coupled, because they are tied to
the availability and liveness of the same process.
