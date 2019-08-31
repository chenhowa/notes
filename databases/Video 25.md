# Crash Recovery and NoSQL

## Crash Recovery

From ARIES, there are three phases:

1. Analysis - reads end_checkpoint to analyze the Xact Table and Dirty Page Table, and recreate an approximation of it at the time of crash.
2. Redo all actions (unless we know they were successfully done), including ongoing ABORTS from the oldest uncommitted dirty page table. 
3. UNDO all failed Xacts

So one of the basic ideas here is that once end_checkpoint is written, we have a snapshot of the dirty pages and active Xacts at the time of the checkpoint. It's fuzzy, in that *some of the dirty pages in the checkpoint may have been written to disk at the time*.

We then examine the log after the checkpoint to try to fill the DPT with all the possible suspected dirty pages we can find, and the Xact Table with all the possible running or aborting Xacts it can find. By the way -- if the buffer manager flushes a page to disk, it also writes a log of that flush.

We then REDO all the logs that need to be redone, starting with the oldest recLSN (the oldest log that might have dirtied a page), and starting from there and moving forward, we check each log, see if it needs to be redone, and if so, we REDO it. We *don't force these REDOs to disk*, but the buffer manager might.

How do we need to know if it needs to be redone?

1. If it's not in the dirty page table, it doesn't need to be redone. Apparently our analysis will definitely detect *all possible dirty pages that existed at the time of crash*
2. If the dirty page table also refers to the same page as the log, but the DPT's recLSN > LSN, then clearly the dirty page was later written to disk, so there's no point in REDOing this particular log.
3. If we check the DB to see the actual state of the page, if the page's pageLSN >= the log's LSN, then we don't need to redo, since we know this log's change actually got flushed to disk.

Then we start undoing Xacts that had not completely committed at the time of crash. The list of ToUNDO starts with all lastLSN's of all the Xacts that are in the Xact Table, starting, with the largest LSN, and each time an UNDO happens, we add the prevLSN of the undone log to ToUNDO, and then we repeat (starting with the largest LSN in ToUNDO). As this UNDOing is happening, we are writing CLR logs to the log so that if there is a crash during recovery, we can continue UNDOing.

### Additional Crash Issues

What if the system crashes during Analysis, or during Redo? It just redoes the recovery protocol.

To limit the amount of work in REDO, you should configure the buffer manager to flush more often (we don't want even hot pages to not-flushed for long times, even with LRU)

To limit the amount of work in UNDO, you need to prevent people from running really long transactions (abort those).

## Distributed Data, Replication, and NoSQL

Why Replicate Data?

* Increase availability
  * Survive catastrophic failure, like the destruction of an entire datacenter in one place.
  * Bypass transient failures -- if a server is temporarily very slow, the request can just be handled by a different server.
* Reduce latency
  * Choose a nearby server
  * Ask many servers, but take only the first response
* Load balancing - no one server is overwhelmed by requests, which would slow the response down.
* Other reasons
  * Security, system management, etc.

### Replica Consistency

Replica consistency is *linearizability of single writes*, which provides the illusion that when a data item X is updated, all copies of it are updated at the same time atomically. In other words, readers will alawys see values of X as if there were no copies at all.

This has nothing to do with the C in ACID, or with serializability, which both have to do with transactions. Replica Consistency is just about single-object-updates.

### Traditional DB Replication

Traditionally: 

* Small number of big databases (one per continent), and replication is for failure-recovery
* Data-shipping - send the data to all the other databases
  * Both databases (sender and receiver) have to do all these random IO's for this. Expensive.
  * Hard to get transactional guarantees for replication.
  * One pro - easy to plug differen systems into this replica system.
* Log-shipping - send its logs to all the other databases
  * Cheap at the source node, since you were writing the log anyways, and the log records are smaller records
  * Transactions are effectively transferred thanks to the log.
  * Tricky to do across different systems, since different systems have different log formats.
    * This often means that enterprise will go with DBMS's from only one vendor.

### Single vs Multi-Master writes

* Single master
  * Every data item has one master node. Writes are first performed at master, and then propagated replicas to other nodes.
  * 2PL/2PC can be used for transactions at the master node
  * BUT: The master node becomes a bottleneck! Bad for reducing latency and load balancing.
* Multi master
  * Writes could happen at any node that has a replica of the data item, and then Replication occurs.
  * Allows lower latency and load balancing.
  * Fast, but hard to understand some things
    * What if the item is updated in two places? Which update wins, or are they additive somehow?
    * 2PL/2PC cannot be used effectively here?

### Compromise: Quorum Writes

Suppose you have N nodes replicating each item. Send each write to a subset W of the N nodes, and timestamp the write. Send each read to a subset R of the N nodes and timestamp the read. The reader will use the timestamps to choose the result:

* Ensuring that |W| + |R| >= N is a "strict quorum", which means that a reader will definitely see the latest write from one of the nodes. (pigeonhole principle)
  * If |W| = |R|, |W| > N/2, then we have a majority quorum
  * If |W| = N, and |R| = 1, then we have a writer broadcast.
* Advantages - not at mercy of a slow master, and it's easy to recover from a failed node.
* Transactional semantics still require 2PC.
  * Is 2PC still too expensive, to hold a unanimous vote? The latency depends on your network, and many other factors in hardware (switches, magnetic disk vs flash) and software (GC [esp. Java], event handlers being written to be very very fast)
  * Transactions across the world are still very slow! It may be better, performance wise, to just live with somewhat-inconsistent data.

The danger of using transactions is that misusing them slows the system down a TON, building up queues of pending transactions. This really only happens for geo-replicated systems at massive scale, however.

## NoSQL

Born from Amazon's product *Dynamo*

* Have a simple key/value data model.
* Replicate data at least 3 copies
* Multi-master - write anywhere, with a timestamp
* Update is acknowledged by a single node, with no explicit replication (replication happens sometime in the background)
* Update is "gossipped" lazily among the replicas in the background ("anti-entropy", "hinted-handoff").
* Reads can consult 1 or more replicas 
  * Quorums can be used to achieve linearizability.

### NoSQL ambiguities

* Most NoSQL systems don't bother achieving linearizability
* Per object, you may not read the latest version, and writers can conflict, which requires conflict resolution and merge rules
  * Last/First writer wins -- but what clock do you use?
  * Merge rules (e.g. Increment, Max)
* Across objects, there are no guarantees, so there are no transactional semantics.

### Eventual consistency

If no new updates are made to the object, eventually all accesses will return the last updated value, thanks to the gossipping protocol. In practice, this has no clear meaning, since there are typically always new updates.

This leads to some problems. There are two kinds of properties people want in distributed systems, that Amazon's NoSQL does not guarantee:

* Safety - nothing bad ever happens - Nope! The DB is never really in a correct state.
* Liveness - a good thing eventually happens - "eventual consistency" never really happens though.

To deal with EC, you either don't worry about it (deal with the real-world consequences in the real world, by say, apologizing), you handle the inconsistencies within the application code, or you use some theories ("monotonicity")

### Inconsistency

How often does inconsistency occur?

* What are the odds of an incorrect read? That determines the cost.

To deal with this issue, the basic idea from Amazon is to not mutate values in the key-value store, but to accumulate logs at each node, and then at check-out time, we union-up all the logs to get the correct result, and 2-phase commit is used to write the results to the key-value stores at the nodes.

## Data and Lessons to Live By

NoSQL and Relational DBMS are 75% the same, and 25% tradeoffs.



