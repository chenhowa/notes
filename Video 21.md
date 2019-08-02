# Advanced Topics in Concurrency Control

## Locking Granularity

Do we lock tuples, pages, or tables? If we lock an entire table, than that can block certain transactions from succeeding without aborting, even though the transactions are updating different rows in the table. If we lock tuples, than someone how is scanning a table while making updates will make tons of calls to the Lock Manager. This is expensive, and the lock table may have to get huge, eating up lots of memory. But if we lock pages, it might not be far enough in either direction.

We'd like to be able to choose granularity based on the transaction. So we need *multiple-granularity locks*.

Note that data containers are nested: Database -> Tables -> Pages -> Tuples. We will now have 3 kinds of locks, at each level - Shared locks, Exclusive locks, and intent locks. To lock an item, the Xact needs the right intent locks at the higher levels in the granularity hierarchy. There are 3 kinds of intent locks:

* IS - intent to get S lock at lower level
* IX - intent to get X lock at lower level
* SIX - intent to get S at current level, and an IX lock at lower level
  * This is useful when you want to scan the table, and then later make an update to one tuple. "UPDATE ... WHERE (pred)"

Each Xact starts at the Database. To get S or IS lock on a node, you must have an IS or IX lock on the parent node. To get X, IX, or SIX on a node, you must hold IX or SIX on the parent node. Finally, the locks should be released in bottom-up order, which ensures that conflicts are detected as locks are released by on Xact while another Xact is attempting to get locks from top-down

### Lock Compatibility Matrix

At each granularity level, what locks are compatible with each other among multiple Xacts (IS, IX, SIX, S, X)? See 15:30.

At the Table level, IS and IS are compatible. IX IX are compatible. IX and S are not compatible. X and IS are not compatible. S and IS are compatible. IX and X are not compatible. SIX and IX are compatible, etc.

*Every commercial database uses this multi-level granularity for locks!*

## Concurrency Control for Indexes

Using 2PL on B+tree pages is not great for modifying them, because splits can bubble up to the root, so a lock on the root locks the whole B+tree.

Instead, we'll use short locks (latches). The idea is that the upper levels of B+ trees just direct traffic correctly, so they don't need to be handled with serializable concurrency.

Also, note that even if the B+tree-structure is correct, it's *incomplete* -- and you want to be able to lock stuff that doesn't exist yet.

### Latches

There are many data structures inside the DBMS where you need mutual exclusion locks

* Pin count on buffer pages
* Wait queue in a lock table
* B+ tree pages

A latch allow this mutually exclusive locking, while it will "unlatch" almost immediately, and very quickly. There are S and X latches for reads and writes. Latches are typically placed in memory, right next to the data they will be latching, so cache locality is great.

#### Basic ideas in latching

* There are several:
  * Latch path
    * Base case - get a latch on the root
    * Induction - get a latch on node N, and then get a latch on the appropriate child of N
    * Problems
      * We're still locking the entire table.
      * Around 4 IO's - to read root, child, grandchild, great-grandchild, and latch. Takes forever. We want to avoid holding a latch when doing an IO
    * Benefits
      * The time of holding the latch is the length of a lookup, not a transaction.
  * Latch Coupling (crabbing)
    * Base - get a latch on root
    * Induction - get latch on child of node N, and release latch on N
    * Benefits
      * Releasing latches is great.
    * Problems
      * When is it safe to release a latch on insert? If you get latch on child C, and release latch on node N, and some other insert causes node N to split while a latch on C is held, we may have a huge problem
      * We still hold the parent latch while trying to get the latch of the child.

### B-link Tree

This is a concurrent B+ tree. Readers set no latches at all. Readers will still set locks on tuples and pages, but no latches.

The basic idea is that while going down the B+tree, the node is split. So just detect the twin and go to it if you need to.

39:00





