# Consular Raft

Consular Raft is an increment to Raft consensus to support coordinated
scaling over multiple Raft groups.

For large data sets, i.e. state machines with large states, it is
desirable to partition consensus into multiple Raft groups. This
results in smaller and more manageable Raft logs and snapshots, and
greater concurrency if writes are spread out across data partitions
and consensus can be processed concurrently across Raft groups.

The approach here is that all the Raft groups must have the same
server as leader. This enables the leader server to process all write
transactions without requiring distributed transactions, and to
process consistent reads without requiring data from other
servers. The motivation is to facilitate a broad set of workloads
including complex transactions and analytics.

NOTE: We understand that Cockroach Labs have also scaled Raft. The
approach here differs from Cockroach Labs in that we require all the
Raft groups to have the same server as leader. Hence we could not
reuse the Cockroach Labs work.

# Approach

We want to minimize changes to Raft. We designate a single Raft group
as leaders and leader candidates, and all other Raft groups as workers
that actually maintain state.

The leader and candidate group is called the senate, and its members
senators. The leader of the senate group is called the consul, hence
Consular Raft. The senate is a standard Raft group, without any
modifications.

The worker groups are modified so that each member holds a reference
to its local senator. The leader of each worker group is always the
member whose senator is the consul.

Elections are held only within the senate group. If a worker requires
an election, it asks its senator to conduct an election.

Heartbeats occur only within the senate group. When a worker needs to
check for a heartbeat, it asks its senator for the heartbeat.
