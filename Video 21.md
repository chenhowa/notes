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

At each granularity level, what locks are compatible with each other among multiple Xacts (IS, IX, SIX, S, X)? See 15:30

20:00









