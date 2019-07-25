# Buffer Management, File Organizations, and Indexing

## Aside: Storage Format Review

Page directories can be device independent, if each page directory entry also contains a device ID, in addition to a disk pointer and a count of the free space.

## Page Replacement Policies

For page replacement policies to work, pinners must unpin very quickly, or else the buffer may contain only pinned pages, and no replacement can be done.

### Least Recently Used

That is, the frame with the earliest "unpinned" time. Works well with temporal locality, but is problematic with sequential flooding -- repeated sequential scans of a file that is larger than the buffer. In this access pattern, every lookup will be a miss.

LRU is also very expensive -- to find the least recently used frame, you would maintain a heap, which still doesn't have a constant cost for doing replacing.

### Most Recently Used

That is, the frame with the latest "unpinned" time.

### Clock Replacement Policy

An approximation of LRU. Arrange frames into a logical cycle (like the numbers on a clock). For each frame, store a "second chance" bit. When pin count goes to 0, turn on the "second chance" bit (reference bit). Second chance indicates it was recently used, and tends to be recently used, and we no longer need to be sorting timestamps.

Then, for replacement:

* for each frame in the cycle
  * if pin_count == 0 && second_chance
    * turn off reference bit
  * if pin_count == 0 && ! second_chance
    * choose the page for replacement
* keep going until a page is chosen

Also: *In between replacements, store the last known clock position for use in the algorithm*

### Differences Between DBMS and OS File System

* Pinning API is present in DBMS but not OS.
* Forcing pages to disk
* Ordering writes exactly
* Adjusting the replacement policy and prefetching, based on the access patterns of the chosen Query Plan.

Therefore the DBMS code, if running on an OS, uses very low-level OS interfaces that avoid the OS "file cache" and OS write timing.

## File Organizations and Indexing

Query optimizer will try to guess how expensive operations are, and chooses the least expensive query plan.

There are many ways to organize files, each with pros and cons:

* Heap files - when access is a file scan of all records
* Sorted Files - when access is retrieval of a 'range' of records, or if the records should be ordered by some search key (column, or column combination).
* Clustered Files (with Indexes)

### Cost Model

* B - number of data blocks in the file
* R - Number of records per block
* D - average time to read/write a disk block
* For now, we are ignoring sequential vs random IO, prefetching, and in-memory costs.

Even with these assumptions, the model is still fairly good.

More assumptions:

* Single record insert and delete
* Only one record matches during equality selection
* For heap files:
  * Insert is always to the end of the file
* For sorted files
  * Files are immediately compacted after deletions (which is expensive to actually do)
  * Selections will be done based on the search key that the file is sorted by.

### Costs of Operations in IO's

* Scan all records
  * Heap - B * D
  * Sorted - B * D
  * Clustered - 1.5 \* B \* D (assuming heap file is 2/3 full, with some empty spaces in the middle)
* Equality Search
  * Heap - (B \* D / 2) = O(B \* D)
  * Sorted - O( log(2)(B) \* D )
  * Clustered - D \* (log(F)(1.5 \* B + 1)), where F is the average number of children in the data structure of the index
* Range Search
  * Heap - B \* D
  * Sorted - O( (log(2)(B) + (# of matching pages)) \* D )
  * Clustered - O( (log(F)(1.5B) + (# of matching pages)) \* D )
* Insert
  * 2 * D (to read it and find end, and then write whole block, since writing is done in whole blocks)
  * Sorted - O( D * (log(2)(B) + B/2)) ( find it, then insert it, and then shift records in all remaining blocks over as part of insertion)
  * Clustered - ((log(F)(1.5B) + 2) * D. Walk down index, read the correct page, insert, and then write it.
* Delete
  * Heap - (B * D / 2) + D ( find it, then delete it and write once\)
  * Sorted - O( D * (log(2)(B) + B/2)) ( find it, then delete it, and then shift records in all remaining blocks over to compact)
  * Clustered - ((log(F)(1.5B) + 2) * D

### Clustered Files (Based on Indexes)

Indexes allow record retrieval by value, for one or more fields. For example, "Find all students where the first name column is "Bob" and the last name column is "Nob".

What is an index? An index is a disk-based data structure that allows fast lookup by value, according to a search key -- a subset of the columns. The search key does not need to be a "key of the relation" (that is, the unique identifier for records in the relation).

The index contains a collection of *data entries* (k, (items)), that associates each search key with *some way to find the set of matching items from the relation*.

### Indexes

What kind of lookups (selections)?

* Selection - based on a key, operation, and a constant
* Equality selection (op is '=')
* Range selections (op is '<', for example)
* More exotic selections, like distances between coordinates, regex, n-dimensional space selections
* There are n-dimensional indexes - R-tree, for example.

### Index Structures

* How are data entries in indexes represented?
  * Actual records (with a key value)
  * Pointers (k, record id of data record)
  * Pointers (k, list of record ids of matches). This is more compact than the previous representation.
* Clustered or unclustered index?
* Single key vs composite indexes?
* What data structure?
  * Tree-based
  * Hash-based

In this class, B+-Trees will be used. It is common to have multiple different indexes per file. The cost is in terms of storage, and performance -- every time a tuple is changed, every index may have to be changed.

### Data Entry Representation - A Closer Look

Storing (k, actual record)

* This serves as a file organization for records, just like heap files and sorted files are.
* One such index per relation, to avoid storing a tuple multiple times.
* No pointer lookups to get data records.

Storing (k, record id), or (k, record ids)

* Multiple indexes per relation
* Storing multiple rids is more compact, but then you have to store variable-sized data entries (not that big of a deal)
* For large rid lists, a data entry may span multiple disk blocks

### Clustered vs Unclustered Indexes

In clustered:

* Data entries are stored in the approximate order of the search keys in the data records. In other words, the data entries have about the same order as the cluster of rids that they point to. This enables sequential IO access when searching by the index's key.
* A file can have a clustered index on at most one search key
* Alternative 1 is clustered by definition!
* Clustering means "mostly sorted"

In unclustered:

* The data entries have rids that are randomly spread throughout the file.
* Doing range searches is thus expensive -- no sequential IO's.

With Alternative 2, we might put all data records in a heap file, then sort the heap file by the search key, and then build the clustered index on the heap file. When an insert occurs in the heap file, the insert will either be random or occur at the end. The clustering will then start to fall apart -- hence, clustering on a heap file is "mostly sorted", until you rebuild the clustered index.

Pros of Clustered Index

* Range searches are fast, if you follow the index
* Possible locality benefits from sequential IO.

Cons of Clustered

* More expensive to maintain the clustering.
  * Either on the fly, or via occasional rebuilding
  * You can leave empty free space in the heap file to absorb some initial inserts while maintaining the clustering.
    * This is why the heap files are usually 2/3 full to accommodate inserts.

### Composite Search Keys in Indexes

Suppose the search key is (age, salary). Then you can query

* age > 24
* age > 24 and salary = 75
* age = 24

But you cannot query "salary = 75", because the index entries sort in lexicographic order. Therefore searching only on the second key "salary = 75" requires a full scan of the file.









