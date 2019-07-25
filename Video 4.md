# Out-of-Core Join Algorithms

Grouping/Aggregation is one kind of rendezvous. Joining is another, since items from multiple files/tables need to be combined.

## Cross Product

In this join (the Cartesian product), from two collections X and Y, generate a new collection Z consisting of all pairs (x, y) where x is from X and y is from Y. The cross product will have |X| * |Y| items.

## Theta Join

Perform a cross product, and then keep only the items that satisfy the "theta predicate"; that is, where *Th*(x, y) is true. A common case of this is the EquiJoin, where the theta function compares two columns in x and y for equality (often these columns are keys)

The EquiJoin allows for an optimization when the Theta function compares keys -- you just need to do one lookup from each table to compare items with the same keys. No need to build the full Cartesian Product first.

## Notation

Let R be a collection; [R] be the number of pages needed to store R, p be the number of records per page of R, |R| be the number of records in R. Then p * [R] = |R|

## Joins

Joins are common, and X and Y are typically very large. So to generate the cross product and then filter with Theta is very inefficient, since we have O( n^2 ) records that will need to be held in memory and disk. We need techniques to reduce this cost.

### Simple Nested Loops Join

* for each x in X,
  * for each y in Y
    * if Th(x, y), then add (x, y)

The cost in IO's is p<sub>x</sub>[X][Y] + [X] = |X|[Y] + [X], because for each record in x, you read in every page of Y and do comparisons against each record in the page, and you have to read every page of X to get the individual items. This cost is way too high -- O(n^2)

### Page-Oriented Nested Loops Join

* for each page b<sub>x</sub> in X
  * for each page b<sub>y</sub> in Y
    * for each record x in b<sub>x</sub>
      * for each record y in b<sub>y</sub>
        * if Th(x, y), then add (x, y)

Cost = [X][Y] + [X] IO's.

This can be improved still further by reading in more pages of X at once before reading in a page of Y, since we have the memory for this. We'll call this *Chunk-Oriented Nested Loops Join*. This has a cost of [X] + (number of chunks of X) * [Y] IO's.

### Index Nested Loops Join

If we are doing an EquiJoin, what if we used an index to quickly look up records that match?

* for each tuple x in X
  * for each y in Y where x and y match according to index lookup
    * add (x, y)

Since lookups in the index can be very cheap, we get a cost of Cost in IO's = [R] + p<sub>r</sub>[R] * (cost to do index lookups). The cost to do the lookup is likely very small, around 3 or 4 IO's.

This can be improved still more using page-oriented or chunk-oriented reads of X, as in the previous sections.

## Back to Iterators - When We Need Asynchronous IO

An example for Sort: Main inits Sort inits FileScan. Then Main calls next and gets tuples. When Sort is done with FileScan, Sort will close it, to ensure the number of allocated resources at any one time is reduced whenever possible.

Suppose now that the data is too big for a Disk, so there are multiple Disks feeding a CPU. How can you maximize the utilization of all the disks so that the disks are never waiting and the CPU is never waiting? This is a *balanced* architecture.

But achieving a balanced architecture is difficult in the single-threaded model of iterators, since only one disk is going to be utilized at a time. So you need multiple threads per disk to be reading from each disk at the same time. This multithreading typically occurs at the FileScan iterator, which allows the other iterators to be single-threaded alone.

## Sort-Merge Join - for EquiJoin

* Sort X on the join attributes
* Sort Y on the join attributes
* Scan sorted-X and sorted-Y to pairs that satisfy the Theta predicate, which is probably an Equality or Comparison function if we want Sort-Merge to work. At this step, you do a little nested loops join over the small range of matching items, to generate all pairs.

Cost in IO's = C(Sort X) + C(Sort Y) + C(Scan)

This does not necessarily improve over Chunk-Nested Loops Join.

We can improve on this algorithm by combining sorting with merging, to remove many IO's from writing the entire sorted files to disk in one place:

* Generate Sorted Runs of X
* Generate Sorted Runs of Y
* Merge and Join X and Y at the same time

How much memory do we need to do this in just two passes (that is, we don't need multiple merges). If we have B blocks of memory, then to do it in two runs, we first generate sorted runs of size at most B on the disk in the initial phase. Then, in the second phase, if we are able to merge them together, the total number of sorted runs cannot be more than (B - 1). So we end up with a maximum number of blocks of B(B-1) again. Since the total number of blocks is [X] + [Y], the memory requirement is sqrt([X] + [Y]).

Sort-merge join also ensures that the output stream is sorted -- query optimizers will care about details like this.

## Hash Join - for EquiJoin

* Generate hash partitions for X
* Generate hash partitions for Y
* Read in hash partition for X and build hash table.
* Stream in hash partition for Y and check for matches in the hash table, and then send theta-tuples to result.

The X partitions have to fit in memory, but the Y partition does not, since it is only being streamed into memory.

Cost in IO's

* Partitioning - 2([X] + [Y]) IO's, since you need to read and write both to build the hash table.
* Matching - [X] + [Y] + [Output] IO's

### Optimizing the Hash Join - Hybrid Hashing

If building relation is smaller than memory, you don't need to partition! It becomes a one-pass algorithm, but then performance becomes a step function.

To avoid the step function, during the partition phase for the building table, you can maintain a single hash bucket in memory that takes up *most* of memory, but leaves enough to continue to partition what doesn't fit in memory. Then during partition of the probing relation, you can stream across the in-memory hash bucket during Phase 0, and then put the remaining partitions on disk, and then do the standard hash join for the disk partitions later. This is an optimization, sure. But not that important.

## Hash Join vs Sort-Merge Join

Sorting is good if input is already sorted. Not sensitive to data skew or bad hash functions.

Hashing can be cheaper, esp. with hybrid hashing, or if the building table is really small.

Query Optimizers will do the IO equations to determine the best algorithm to use for the given tables.






