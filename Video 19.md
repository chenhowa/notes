
# Transactions

ACID

* Atomicity - all actions happen, or none happen
* Consistency - if the DB starts consistent, it ends consistent after the Xact.
* Isolation - Execution of one Xact does not interact with the other Xacts
* Durability - if a Xact commits, its effects persist

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

* Xact must obtain a S (shared lock) before reading, and an X (exclusive) lock before writing. Xact can acquire locks well in advance of when they use it.
* the Xact cannot get any new locks once it has released at least one lock (two phases: 1. Acquiring locks, 2. Releasing locks)
* Lock compability - if an object has an exclusive lock on it by some transaction T, no one else can get any sort of lock on it. If an object has a shared lock on it by some transaction T, any other transaction S can also get a shared lock on it, but not exclusive locks. This is essentially the same matrix that is used by Rust's Ownership system. However, we will need some policy for handling *deadlock* while the locks are being acquired

You can see that this enforces conflict-serializability, since it enforces a certain order on the transactions. But it does *not* prevent *Cascading Aborts*.

### Cascading Aborts

Problem - If two transactions T1 and T2 have interleaved actions, then if T1 gets aborted, then T2 may have to abort (if it reads data that T1 wrote). The fix is to use strict two-phase locking -- a transaction must release all locks at once, whether at commit time or at abort time. This means that some interleaving no longer works. Hence, strict two-phase locking is most commonly implemented, but it only allows a subset of the conflict-serializable schedules.

### Lock Management

Lock manager handles locks and unlocks. It is a hashtable that is keyed on the names of objects being locked and unlocked. The manager keeps an entry for each currently held lock; and the table values contain

* Set of xacts that currently ahve access to the lock (in the case of shared locks)
* Type of lock (shared or exclusive)
* Queue of waiting lock requests

When a lock request arrives, if no other xacts have conflicting lock, then the lock is granted and a new hashtable entry is made (or an existing one is updated). If there is a conflict, put the requestor into the wait queue. A xact with shared lock can request an upgrade to the exclusive lock.

### Deadlocks

A deadlock is a cycle of transactions that are waiting for locks to be released by the other(s). Ways to deal:

* prevention (non-starter, it will occur with databases)
* avoidance
* detection
* timeout (abort if it is taking too long) - but this might abort some transactions that aren't in deadlock, and it can't prevent repeated deadlocking on the same condition

Deadlock prevention is a common technique in operating systems. By ensuring that locks are acquired in a specific order, cycles are impossible. But this *can't be done in a DBMS*, because there are too many objects in a database (tables, tuples, etc.). What ordering would even be appropriate? In addition, getting a lock becomes difficult when doing a JOIN requires an order that goes against the lock ordering. And the interactive user shouldn't be forced into an order.

Deadlock detection is done with a graph of *what transactions are waiting for whom*. We periodically check for a cycle in the graph to find deadlocks. And if a cycle is found, pick a node and abort it.

Deadlock avoidance is doen by assigning the priority of transactions according to timestamps, and then

* wait-die - if Ti has the higher priority, then Ti will wait for Tj. If not, Ti just aborts.
* wound-wait - if Ti has higher priority, then Tj aborts, otherwise Ti waits

This guarantees deadlock avoidance because it enforces an ordering on the lock acquiring. Important detail - if a transaction re-starts, make sure it keeps original timestamp, to make sure it gets older over time, until it gets a high enough priority to run. It's also good to let older transactions finish, since they've likely done more work (locks, writes, etc.), and we don't want to undo all that work.

In the end, deadlock detection is one that is most commonly implemented today.


