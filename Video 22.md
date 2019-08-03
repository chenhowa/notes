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

35:40














