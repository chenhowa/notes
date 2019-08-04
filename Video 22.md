# More on Advanced Concurrency Control

## More on Phantoms

Imagine there are two transactions, T1 and T2, with the following schedule:

* T1 and T2
  * T1 Query in range
  * T1 Uses index
  * T1 Read 10 tuples (no table lock, because of reading only a few tuples using the index)
  * Insert something T1 should have read
  * Repeat read -- now gets 11 tuples. This means that whatever T1 does with this info may now be different from what it did before, unexpectedly. This is the cost of not requiring a lock on the entire table for performance reasons.

  Next-key locking prevents the above scenario

 ## MultiVersion Concurrency Control (MVCC)

 This is an alternative to 2PL. We will look at the Timestamp Ordered MVCC (TO-MVCC).

 Every transaction gets a unique timestamp ts upon entry in the DBMS. For each data item in the DB, keep a set of Transaction ReadTimestamps (R-ts), and keep a set of Transaction Versions (W-ts, version content)

 Then, to do a Read(x), read the version of the data with the largest Write Timestamp that is *less than your transaction's timestamp*, and then add your timestamp to the set of R-ts for the data item

To do a Write(x, value), check the interval from your transaction's timestamp to the smallest timestamp *greater than your timestamp* in the W-ts for the data item. If there was a R-ts for the data item in this interval (someone who read a previous data version, instead of you), then abort your transaction -- you were too late, and re-enter the transaction in the system so that it gets a later timestamp. If there was no reader, then create a new W-ts with your value, for the data item.

So you can see that transactions are ordered by the time they were entered into the system, and this helps make the transaction model very simple

There should be a garbage collection model for this so the R-ts and W-ts don't fill up memory completely

TO-MVCC is very interesting, in that it enforces serializable schedules using the timestamps of the transactions. 2PL aborts things less, but requires locking. TO-MVCC doesn't require locking at all.

Pros:

* More concurrency than 2PL - more readers can go through
* Read-only transactions don't need locks, which optimizes the fast path for reads
* Writes are append-only -- they are conflict free.
* No communication/communication between transactions, which is great in Distributed contexts

Mixed:

* You can't update in-place. There are multiple versions, potentially in multiple places.
  * Distorts data locality on disk (clustered indexes need locality)
  * Have history of data next to the data. Ask questions about data across time.
  * Hard to make TO-MVCC efficient on magnetic disks.

Bad:
  * Garbage collection of versions
  * Restart is expensive, maybe more expensive than waiting for a lock.
    * Wasted compute resources

Recently, Flash SSD is making data locality less important due to random IO being fast. For in-memory databases, lock tables are too slow; removing locks with TO-MVCC maybe much better. Storage is cheaper, so maybe version history can be kept now (is this how ServiceNow does it?)

## Distributed Concurrency Control

How do we get transactions to work if each transaction comes from a diffeerent computer, one that handles updates and inserts? When a transaction comes in, it is assigned to a "coordinator" node. Then where does the concurrency control state (lock table) go?

Keep the locks with the data they are locking (on the same machine). Keep in mind that in this distributed model, the data is partitioned over multiple machines, with *no replication*. That's why this scheme can work.

If each node is doing its own concurrency control, however, somethings can go wrong:

* replication becomes a problem -- replicas may have different locks. But you *could just lock all the replicas the same way*. So assume no replication.
* one transaction can update multiple nodes -- but then, if you abort in one place, or commit in one place, you must abort/commit everywhere. *Making sure that all nodes agree is key*
* Machines can fail - what do you do?
* What do you do with a deadlock? If the plan is to detect deadlock, how do you do it in a distributed system when the "waits for" graph is also distributed across multiple nodes? How do we build the graph? Just send the pieces of the "waits for" graph to one place to be built.

How do we decide to commit/abort when there maybe failure/delay of a node or message? Well, we have a vote, one per machine:

* But how many votes does a commit need to win?
  * Everyone must agree -- otherwise the commit might not be atomic (all or nothing) -- if one of the nodes failed, a piece of the transaction might be lost. Or a node might detect the commit is not consistent.
* How do we do distributed voting?

### 2-Phase Commit (2PL)

* Phase 1
  * Coordinator tells the participating nodes to "prepare" to vote.
  * Participants respond with yes/no votes (unanimity is required for yes).
* Phase 2 - if results are clear, coordinator dissemiantes results to participants.
* If there are failures or delays, logging needs to be done to resolve the vote
  * The coordinator could fail
  * Some other node could fail

### 2PC + CC

If messages are ordered as in Strict 2PL, everything is great-- abortion leads to no cascading aborts of transactions. But this is time-consuming because you run at the speed of the slowest participant (but that's just the price)


## Weak Isolation

If transactions for ACID are too expensive performance-wise, what else could be done? Answer: Weak Isolation

Sometimes transactions are too restrictive - what if one person wants just an approximate average of all grades, but during that, professors can't make individual updates to grades.

Sometimes transactions are too expensive -- computers are waiting for each other.

What can be done? A looser transaction is harder to understand and work with, but it does exists.

### SQL Isolation Levels (Lock-Based)

A terrible explanation from the Professor. Not enough details, but here you go:

* Read Uncommitted
  * "It's okay to read dirty data (uncommitted writes)"
  * Implementation -- no locks before reading.
* Read Committed 
  * "You can only read committed items"
  * Implementation -- lock, read and then can unlock immediately after read. Short read locks.
* Cursor Stability
  * "Make sure reads are consistent while the application is paused".
  * Implementation -- lock what the application has just read, and then unlock when application is done with it.
* Repeatable Read 
  * "If you read a tuple twice in a transaction, you see the same committed version.
  * Implementation -- hold read lock until the end of the transaction. Long read and long write locks.
  * This gives no protection against phantoms
* Serializable
  * Long read locks, long write locks, and phantom protection.

It's hard to reason about what happens with these, under various scenarios, so most people just don't worry about it. Does ServiceNow do this?

### Snapshot Isolation

All reads made in a transaction are from the same point in (transactional) time; typically the start time of the transaction. This is to say, if the transaction starts at 4 PM, all reads will reflect the state of the data at 4 PM

If any of the transaction's writes conflict with any writes since the snapshot was made, the transaction should abort.

But this is not equivalent to serializability, even though Oracle (the pioneer of this method) claims that it is.

#### Problem with SI - Write Skew

Suppose you have a Checkings (C) and Savings (S) account, where the constraint is that Ci + Si >= 0. Suppose C1 = S1 = 100, and that transaction T1 withdraws 200 from C1 and T2 withdraws 200 from S1. In a serial schedule, this will fail due to the constraint. But in Snapshot Isolation, both T1 and T2 may think that C1 + S1 = 200, since they only have their own snapshot. So the end balance will be S1 + C1 = $-200

So this happens when you have a constraint that stretches across multiple objects, like above.

### Defaults

Most DBMS don't ship with Serializable Schedules as the default. Most ship with some sort of weak isolation (and therefore non-ACID transactions)

## Logging and Recovery

* Atomicity - all of a transaction or nothing
* Consistency - integrity consistents
* Isolution - Xact is isolated from others
* Durability - if a Xact commits, its effects persist

But for atomicity, transactions may abort (or fail). So we have to be able to roll all the changes back.

And for durability, even if the system crashes (say the power goes out, or the disk corrupts, or the CPU dies, or the DBMS is not available, or user error, or software bug crash), so long as the commit occurred before the crash, it must persist

So assume that Concurrency Control is working (Strict 2PL, in particular), that updates are happening in place (flushed to DB), with no private copies.

We could do one transaction at a time, serially. On abort -- just don't commit. The database hasn't changed. On commit -- write to db (but what if failure during the write? You need to log the intended write). But this simple scheme isn't performant.

*The buffer manager is hugely important here*

* Force policy - make sure every update is on disk before commit -- provides durability without logging (but poor performance d/t many random IO's)
* No steal policy - don't allow buffer-pool frames with *uncommitted updates* to overwtie *committed data* on DB disk. In otherwords, as part of the replacement policy, a dirty frame can't be written to disk and taken out of the buffer pool if it has uncommitted data.
  * This ensures atomicity without UNDO logging
  * But again, possibly poor performance - the buffer pool may fill up with dirty pages, which may force IO's if all the other pages are pinned.

What iwe want is a Steal and No-Force policy, because we *want high performance*, even though it's complicated to implement.

* NO FORCE - commit may not lead to imemdiate write.
* STEAL - Buffer Manager can write any dirty page to disk as part of its replacement policy. It doesn't care about transactions.




