# Out of Core: Sorting and Hashing

## Why? Sorting

Eliminate duplicates, or summarize groups of items. Thus, sorting leads to rendezvous to bring like items together. It also orders, obviously. Sorting is the foundation to other algorithms, like sort-merge and indexing.

Sorting has to be out-of-core, because large files are too big for memory. Multiple page faults will lead to multiple I/O's, and performance suffers. We want caching, but the I/O's prevent that.

## Disks and Files

Many databases still use magnetic disks, which are slow.

- No pointers, just READ/WRITE of blocks
- API calls are expensive to execute
- Consequently, the way that DBMS's call these API's needs to be carefully planned.

Persistent languages, which store data on disk automatically, have not performed well. Hence, we are left with the disk and file system API's, which requires discipline.

### Economics

Although Disk is an order of magnitude cheaper than SSD, SSD may still be better due to the performance and the problem size.

### Storage Hierarchy

The more performant storage is, the smaller it is. Cache, RAM, SSD, and Disk. Sometimes Disk is the database. Sometimes SSD is the database. Sometimes SSD is just the cache for the Disk.

### Components of a Disk

Multiple platters on a spindle, making a cylinder. Multiple disk arms, with one head on each platter, in the same position. Typically only one head is reading/writing at the same time. Usually only a fixed number of sectors of data, of fixed size.

Costs for rotation of platters and for seek movement across tracks.

- seek time (moving the head to another track) is 2-4 msec
- rotational delay is 2-4 msec
- transfer time - 0.3 msec per 64 KB

When using disk, use software solutions that reduce or amortize the seek/rotation delays. Promote sequential IO's over random IO's.

Sequential blocks lie on adjacent sectors on the same track, followed by blocks on the same cylinder, followed by blocks on the adjacent cylinder. There may be a track cache that holds an entire track, so that even if the processor gets distracted, the track can still be accessed without waiting for rotation.

### Notes on Flash

Single reads are smaller and very fast. Reads are fast, even in random-access.

Write is slower for random (due to need to clear blocks before writing pages, and moving blocks to spread out the wear), but still pretty fast.

Performance and behavior is more consistent than with disks.

### Database Numbers

Today, most significant DBs are not that big -- hundreds of GB at most, usually. Remember to size your techniques to your problem sizes.

## Streaming - Double Buffering

When an input or output buffer is doing IO, switch to a second (or even third) buffer that is either full or empty, so the IO delay is minimized. Hence, streaming can be quite fast.

## Sorting and Hashing

File F takes N blocks of storage space. We have B blocks of RAM, and two disks with plenty of space.

In sorting, F will be transformed into another output file FS, which is sorted on disk.

In hashing, F will be transformed into one or more files, where all things with the same hash value are stored in the same unordered partition.

### Sorting: 2-Way

Use the streaming algorithm to read/write from disk.

Pass 0 (conquer)

- read a page, sort it, write it to a new file. Repeat.

Pass 1, 2, 3 ... (merge)

- read from two sorted files of data, merge, write to a new file. Repeat.

How many passes? log(2)(N) + 1 (initial sort)

For each pass, each block is read/written (possibly sequentially). So the total cost is O( N log(2)(N) )

### Sorting: B-Way

Same as 2-Way, but this time we use way more input buffers for pass 0, and way more input buffers for pass 1.

Number of passes: 1 (initial sort) + log(b-1)(N/B)

### Memory (RAM) Requirements for Sorting

After Phase 0, each sorted run is of size B. After Phase 1, if we are done, then there must have been only (B - 1) sorted runs. Hence the total file size is on the order of B^2 blocks. So to sort a file of N blocks, memory should have sqrt(N) blocks.

### How to do Phase 0 of Sorting faster

QuickSort is a fast way. HeapSort is also available, while using two heaps. See ~43:00

HeapSort can get lucky and have very large runs, so very large sorted sub-files for the later merge. Quicksort cannot do this. But QuickSort has better cache locality in memory. So sometimes QuickSort is better, sometimes HeapSort is better, but QuickSort is more popular.

### Hashing

We use hashing when we don't need order, but we do need grouping into partitions. Like for removing duplicates and forming unordered groups. It often just requires "rendezvous" matches. And hashing may be cheaper than sorting.

Phase 1: Use a hash function h to create rough partitions on the disk, through a streaming algorithm with one input buffer and multiple output buffers. (one partition may include all apples and strawberries, while another may contain all oranges and grapes)

Phase 2: If a partition fits in memory, read all of the partition into memory and build an in-memory hash table there.

Notice how important it is to choose a hash function that correctly and completely separates *groups of groups*!

Cost is 4N IO's -- read/write all blocks to divide into partitions, and then read/write all blocks for the in-memory hashing.

Phase 3: If a partition doesn't fit into memory, and can't be further partitioned, then hashing will not work. However, if the partition can be partitioned again with a second hash function *g*, then we can succeed after some *recursive partitioning*.

### Size of table that can be hashed in two passes

There are (B-1) partitions, with N/(B-1) blocks in partition, at most. To read all of it into B blocks of memory, the total number of blocks is at most (B - 1) * B. So again, memory requirements is B = sqrt(N)

### Comparing Out-of-Core Sorting and Hashing

Sorting and Hashing are flip sides of the same coin, from an IO perspective.

Sorting is conquer and merge, while hashing is divide and conquer. Hashing is random in, sequential out. Sorting is sequential in, random out. It's interesting that they are dual from an IO perspective.

Remember that hashing is sensitive to skew -- it may not work if the data partitions cannot be made small enough. Sorting is not typically sensitive to skew.

Both can allow for in-memory performance tuning (through cache locality?)

Hashing scales well with the number of rows, not the number of items (example: counting males and females in 1 TB of data) ~ 1:04:30

### Parallelizing Sorting and Hashing

To parallelize hashing, add an initial step to roughly partition data across machines first, by splitting them up through the network. Then you can run the regular hashing algorithm.

To parallelize sorting is much harder. First, partition sort-wise across machines in the first step. That is, 0 -> N should go to Node 1, then N -> 2N should go to Node 2, and so on, which is really hard to do well. Skew across machines is very likely -- one node may get much more data. But if you are able to do this (perhaps by some random sampling of the data, and statistics), you can merge all the machines together after.















