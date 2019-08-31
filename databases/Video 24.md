# More on Crash Recovery

We want *Steal* and *No Force* policies for Crash Recovery.

* No force - how do we make sure we can REDO if the system crashes while a committed transaction is being written to disk.
* Steal - We need to be able to UNDO if an Xact aborts, or if the system crashes before the Xact has committed.

## Logging

For every update, we will record REDO and UNDO information in a log, which is on its own disk. The writes will be sequential, and minimal info (just the diff) will be written to the log.

Thus, the log is an ordered list of REDO/UNDO actions, and each log's records contains (tranasaction id, page ID, offset, length, old data, new data ), and some additional control info.

### Write Ahead Logging

The basics of the protocol:

* The log must be on the disk before the corresponding data page can be written to the disk.
* Before an Xact can be committed, all the log records to a Xact must be forced to disk.

With the two above protocol points, the ability to UNDO and REDO are guaranteed

Each log record can be in the log, in the database, and in the buffer pool in RAM. To link them together, we have a unique id for each log record called the "Log Sequence Number", which is always increasing (strictly monotonic).

Each data page contains a pageLSN, which is the Log Sequence Number (LSN) of the most recent log record that made and update to the page. The Database System in memory keeps track of the flushed LSN, which is LSN of the log record that was most recently flushed to disk.

Logs must be written to disk *in sequential order*.

Finally, before page i can be written to the DB, it muts be true that its pageLSN must be less than or equal to the flushed LSN. This guarantees all the log records for this page have been flushed to the DB -- this policy itself enforces the Write-Ahead Logging.

### Log Records and State

* LSN
* prevLSN - the LSN of the previous log record written by the log record's transaction.
* XID
* type - update, commit, abort, checkpoint (for log maintenance), compensation log records (CLRs), end (of commit or abort)
* update record fields (pageID, length, offset, before-data, after-data)

In memory, there are two in-memory tables:

* Transaction Table
  * One entry per active Xact, which contains the XID, Status, and lasLSN (last log generated by the Xact; may not be in memory). Note that with teh lastLSN, you can walk backward along the entire chain of logs generated by the Xact.
* Dirty Page Table
  * One entry per dirty page in the buffer pool. Contains recLSN - the LSN of the log record which __*first*__ caused the page to be dirty

### Xact Execution

Normally, an Xact is a series of reads and writes, followed by commit or abort, using Strict 2PL and STEAL, NO-FORCE buffer management and write-ahead logging.

During transaction commit:

* Write a commit record to log
* FLush all log records up to the commit record to the disk, in order.
  * This guarantees that the system's flushedLSN >= lastLSN of the transaction, so all logs for the Xact are on the disk (done in sequential order).
* Then write an *end* record to the log

Note that not all the data pages have been written to the DB at this point! The log itself is enough, because we are letting the Buffer Manager decide when the data pages are actually flushed.

### Xact Abort

Suppose Xact is aborted (no crash). To play back the log in reverse order, UNDOing updates

* Get lastLSN of Xact from the Xact table
* Write an Abort log to the end of the log
* Follow the chain of log records backwards using the prevLSN field of the log
* For each undo, write a compensation log record (CLR) for each undo, that notes that we are doing an undo.
  * A CLR has one extra field - "undonextLSN", which points to the next LSN that will be undone (the prevLSN of the source log record).
  * Thus, the CLR contains REDO info for the UNDO, in case there is a CRASH during the abort -- want to resume the abort.
  * CLR's are NEVER undone, but they might be redone. Applying UNDO's is not idempotent -- doing an UNDO twice might do the wrong thing.
* After all the UNDO's, write an "end" log record, and then flush all the UNDO log records to disk (up to the LSN of the log record) -- *then the undone data pages can be written to disk, which is the write-ahead logging*. Note that this is why we need the dirty page table -- the buffer manager has to check whether the corresponding log record has been written to disk before it can flush the dirty page.

Note that to perform UNDO, we need a lock on the data, which is fine because we still have 2PL locks.

### CheckPointing

With the log, we essentially have the history of the entire DB. So periodically, the DBMS creates a checkpoint to minimize recovery time after a crash. To create a checkpoint without copying the entire DB:

* Write a begin_checkpoint log
* Write an end_checkpoint log that contains the current Xact table and Dirty Page Table and *flush to the log*. This is a fuzzy checkpoint.
  * The checkpoint is only acccurate as of the time of the begin_checkpoint record, since other Xacts continue to run during checkpointing.
  * The effectiveness of the checkpoint is limited to the oldest unwritten change to a dirty page.
  * Store the LSN of the most recent checkpoint record, in some sort of conventional place.

### Crash Recovery

* Start from the master record's checkpoint.
* Analyze to figure out which Xacts committed since the checkpoint, and which failed.
  * You do this by reading the log from *begin_checkpoint* to *end_checkpoint*, and seeing what Xact logs were written in between *begin* and *end*, to generate an accurate Xact table and Dirty Page Table, and now you know which transactions were committed, and which failed, during the crash.
* REDO all actions.
* UNDO all failed Xacts.

As you go through the log in the checkpoints, we update the the corresponding Xact in the Xact Table with its lastLSN and status, based on the log. Similarly, we can reconstruct the Dirty Page Table - each page's recLSN is the first update to the page found within the checkpoint -- Does that actually make sense?

The oldest thing that needs to be redone, is the oldest thing that never made it to the DB, even though it should've. This is the oldest recLSN in the reconstructed Dirty Page Table. We will redo everything starting from, and after, that particular log, to get the DB to the state it was in during the crash. Then we can UNDO all the Xacts that failed (were not committed during the crash) thanks to the crash.

#### The Analysis Phase.

* Re-establish state at checkpoint through the stored Xact table and Dirty page table. Then scan the log from that checkpoint --
  * If we see a "end" record, remove that Xact from the Xact table, and MAYBE remove its pages from the Dirty Page table?? -- we have all the info we need to flush all the Xact's changes to disk when we REDO, and we don't need to worry about correctly undoing it.
  * For all other logs - add the Xact to the Xact table if it was absent, and set lastLSN = LSN. If the log was a "commit" log, also change the Xact status. If it was an "update" log, if the page P was not in the Dirty Page table, add P to the Dirty Page table, and set its recLSN=LSN

By the end of analysis, the Xact table says which Xacts were active during the according to the last log flush before the crash, and the DPT says which dirty pages *might not* have made it to the disk.

*The important thing is to be able to undo all the failed Xacts, including partially committed Xacts that didn't manage to write an "end" record* The fully commited Xacts don't need to be touched.

#### The REDO Phase

Scan forward from the log record containing the smallest recLSN in the DPT. For each update log or CLR with a given LSN, REDO the action unless 

- the affected page is not in the DPT - for some reason, according to analysis. How might this happen?
- affected page is in the DPT, but has recLSN > LSN, which means this update is in the Database, thanks to the buffer manager flushing it before it got dirty again.
- pageLSN (in DB) >= LSN, which requires IO, is another indication that the update did get put to disk.

To REDO the action, reapply the logged action to the page in memory, and set pageLSN to the LSN. No logging, and these changes are not forced to the database

#### The UNDO Phase

We could undo each Xact still in the Xact table, and abort all of them. The problem is that this is slow, since you have to do random IO's for possibly many, many Xacts, in random order.

To improve the performance, gather the lastLSN's of all the Xacts in the Xact table, and 

* choose the largest LSN that needs to be undone and
* If it is a CLR and undonextLSN == NULL, write an End record for this Xact.
* If this LSN is a CLR and undonextLSN != NULL, then add undoNextLSN to the list of LSN's to undo.
* Else it is an update -- undo it, write a CLR, and add the *prevLSN* to the list of LSNs to undo.
* Repeat until the list of LSN's to undo is empty.

The goal of this is to do the undos in just one sequential backwards pass of the log. Note the beauty of the CLR's -- it means that even if an UNDO is being interrupted, it can be finished later (even after a crash)

This lecture was pretty terrible, in that it wasn't illustrated very well.
