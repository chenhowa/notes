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
  * Fast, but hard to understand soem things
    * What if the item is updated in two places? Which update wins, or are they additive somehow?
    * 2PL/2PC cannot be used effectively here?

### Compromise: Quorum Writes

55:00




