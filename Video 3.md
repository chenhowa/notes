# Single Table Queries

## Relational Tables

Provide a schema - types and names. A table's schema is generally fixed, although it can be changed. It's more typical for rows to be updated. Each table contains a multiset of rows.

## Software Architecture Behind Queries

- SQL Query is fed into
- Query Optimization and Execution, which provides a Query Plan bytecode as
- Relational Operators, which utilize
- Files and Access Methods, which improve performance with
- Buffer Management (in-memory cache of disk blocks), which is built above
- Disk Space Management, which reads/writes from the Disk (very much like the File System in Operating Systems)

Additionally, thanks to multiple users and possible crashes, we need software for Concurrency Control, logging, and recovery for File Access, Buffer Management, and Disk Space Management.

In the past, OS did not manage buffers and disk space appropriately. The situation is somewhat better today.

### Query Optimizer

Query optimizer translates SQL into an internal language of "relational operators" that the data (tuples) flow through - a Query Plan, which is optimized to be executed. This allows the syntax of SQL to be changed without affecting the code generation.

Some possible operators include

- File Scan
- Sort
- Distinct

### Interface of Relational Operators

The interface strongly resembles "iterator", with three methods:

- init, which initializes state (with references to inputs, especially)
- next, which returns the next tuple based on its internal input state (return may be over the network)
- close, which cleans up and finishes

Each relational operator has one or more inputs from which it generates tuples for the next operator in the pipeline.

This interface-based system means that the operators can be arranged in almost any order, easily. Good software engineering technique.

### Sort Iterator

- init generates sorted runs on disk
- next begins merging and passing the next sorted item to the caller (which may or may not be over the network)
- close destroys the allocated runs on disk

Notice that in the iterator model, data flow is synchronous. Iterators are not running asynchronously. Async introduces overhead, but sync is not good for multiple expensive relational operators that you would want to run async to improve performance and maximize CPU utilization.

### Aggregate and Group By - Implemented with Sort

Sort iterator will order by the group by columns. The Aggregate iterator will then track which group it is currently in, and compute the aggregate for that group, before finally passing the aggregate result to the caller.

For example, for COUNT, Aggregate will track the "count-so-far" of the current group; for SUM, it will track "sum-so-far", and for AVG, it will track both "sum-so-far" and "count-so-far" for the current group. Once the current group ends, it will return "sum-so-far"/"count-so-far". *Notice that for each Aggregate result returned, multiple tuples from the Sort iterator are consumed*.

### Aggregate and Group By - Implemented with Hash Naively

Hashing implementation is almost identical, except the groups don't come in sorted order.

### Aggregate and Group By - HashAgg

To get a win, we can do the aggregate during Phase 2 of hashing. As each group is read in from the partitions on disk, instead of building a hash table from the hash to the records, we build a hash table from the hash to the aggregate value (like the count). This saves memory (likely lots) and reduces computation time. This line of thinking can lead to a huge optimization on the memory costs of generating the aggregate, since it means that in HashAgg, the size of each partition no longer has to fit in memory, since the partition isn't being stored in memory -- only each aggregate values are -- likely to be small in memory.




