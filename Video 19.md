
# Transactions

ACID

* Atomicity - all actions happen, or none happen
* Consistency - if the DB starts consistent, it ends consistent after the Xact.
* Isolation - Execution of one Xact does not interact with the other Xacts
* Durability - if a Xact acommits, its effects persist

## Isolation

This is part of concurrency control. We define serial schedules for Xact, where each Xact runs one after the other. This is safe, but slow, but it defines what correct behavior really is.

* Serial Schedule - each Xact runsf rom start to end without other Xact interfering
* 2 schedule are equivalent if that ahve the same DB actions, and leave thev DB in the same final state (given an input state)
* A schedule is serializable if S is equivalent to any serial schedule

### Conflicting Operations

Two operations conflict if:

* they are from different transactions
* they are on the same obejct
* at least one of them is a write

Two schedules are conflict equivalent

* They involve the same actions of the same transactions
* Every pair of conflicting actions is ordered the same way (in terms of which runs first)

A schedule S is conflict serializable if S is conflict-equivalent to some serial schedule. *Conflict-serializable schedule are easier to identify and enforce that just serializable schedules*. A schedule S is conflict-serializable if you can swap consecutive non-conflicting operations from dfiferent transactions in the schedule to produce a serial schedule

#### Dependency Graph

There is one node per Xact, and there is a directed edge between nodes if one operation from Ti conflicts with an oepration from Tj (the node with the earlier operation is the start of the directed edge). Then, a schedule is conflict-serializable if and only if its dependency graph is *acyclic*. So building this graph, and checking for cycles, is how we determine whether a schedule S is conflict-serializable

### View Serializability

This is a weaker notion of serializability than the earlier versions. Schedules S1 and S2 are view-equivalent if 

* They have the same initial reads (reads before they've been written in the transaction)
* They have the same dependent reads (if Tj writes a value A, and then Ti reads it, the read is a dependent read)
* Same winning writes (if Ti writes final value of A in S1, then Ti also writes the final value of A in S2)

View serializability includes *all conflict serializable schedules* as well as schedules with "blind writes" (writing to objects that are done without reading the object first). But view serializability is harder to implement than conflict serializability, which can be enforced efficiently

## Implementing Conflict-Serializability

We will use the two-phase locking (2PL) scheme, the most common scheme. It is pessimistic - you set a lock on an object before you read or write it, to avoid conflict

* Xact must obtain a S (shared lock) before reading, and an X (exclusive) lock before writing.
* the Xact cannot get any new locks once it has released at least one lock (two phases: 1. Acquiring locks, 2. Releasing locks)
* Lock compability - if an object has an exclusive lock on it by some transaction T, no one else can get any sort of lock on it. If an object has a shared lock on it by some transaction T, any other transaction S can also get a shared lock on it.

You can see that this enforces conflict-serializability, since it enforces a certain order on the transactions. But it does *not* prevent *Cascading Aborts*

33:00







