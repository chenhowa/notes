# Storing Data: Disks and Files

## Aside: Hashing Questions

If your Hash function is unable to sub-partition until the partition is of size B, the implementation should recognize that, throw an exception, and then use the Sort function instead.

## Definitions

Block/Page is a unit of transfer for disk read/write. 64 KB is a good number for today.

Relation = Table

Tuple = Row = Record

Attribute = Column = Field

DBMS store info on disks or SSD, and reading/writing costs are more expensive than the corresponding RAM costs.

* seek time - moving disk head to right track.
* rotational delay
* transfer time - time to read/write onto disk from head

We can minimize these by pre-fetching information, especially during a sequential scan, by grabbing several pages at a time into the disk buffer, so that rotational delay is eliminated for later reads. Hence sequential IO is much, much faster than random IO.

## Disk Space Management

This is the lowest layer of the DBMS, and manages space on the disk. Rather than write directly to a device, the database is allocated as a sequential file. It can allocate/de-allocate a disk page, and read/write a page.

Higher levels will often assume that the disk manager is performing well -- that sequential pages are stored in sequential order, and come quickly.

File - a collection of pages, each containing a collection of records. File abstraction must support insertion/deletion/modify/read record, and scanning all records (possibly with some filter function). By contrast, in UNIX, a file is a stream of bytes.

*A single file is typically implemented as multiple OS (Unix) files, that may span multiple disks*. Or it can be implemented as raw disk space, if you're just writing to file.

## Unordered (Heap) Files

* Collection of unordered records
* Disk pages are allocated/de-allocated as file size changes
* Supporting record level operations means we need to track
  * pages that are in the file
  * where free space in the file is
  * where records are

### Heap File - Doubly Linked List of Segments

One list of full pages, one list of pages with free space, and one header page with an id, and the heap file name, that can be found through a standard database catalog somewhere. Each page has a pointer to the previous page, and a pointer to the next page, in its list. These pointers consist of the correct block number on the disk.

Problems:

* Don't know how much free space is available on which page, without walking the lists in search of that -- at least, not without caching it in memory.

### Heap File - Page Directory

At the front of the file, have a bunch of header pages, that store a pointer to a page along with the free space count for that page. Multiple (page, metadata) tuples can be stored in a header page.

You can build directories of directories for more organization, if needed. The directories can then be cached in memory for quick access.

Note: We can build indexes on heap files so that records can easily be fetched by the value of one or more columns, instead of by record id.

### How is a Row Stored on Disk? Good Question

If all the fields have a fixed length, then the field values can be stored sequentially on the disk, starting from some base address. It's easy to find the *i*th field by doing some arithmetic from the base address.

But if records have variable length fields, we have two formats:

* Use delimiters to mark the end of fields. But this can slow down processing, since you have to check for the delimiter all the time.
* Store field offsets at the front of the tuple to make it easy to find the field. End of record pointer (that is, record length) is stored at a higher level. Null can be represented by having two pointers point to the same offset.

This helps us find tuples in RAM.

### How to Store Multiple Records on a Page

For fixed length records:

* Store records one after the other
* At end of page, write the number of records on the page.
* You can easily compute the amount of free space and the location of each record.
* Deletions can be compacted (packed free space format); or, if you want unpacked, have a free space bitmap at the end of the page.
* Record id is a (page id, slot #) tuple.
  * Problem with compacted is that if you have records for free space management, the Record ID changes. The Unpacked format ensures that the Record ID does not change.

For variable length records, use the slotted page format:

* Store number of records (number of slots) at end of page.
* Store pointer to free space at end of page.
* Have pointers to starts of each tuple on the page (slot directory). The number of pointers is the number of slots
  * This indirection lets you move records on the page without changing the (page id, slot #) tuple.
  * This makes compaction easier to do (we'll do it occasionally to reclaim free space).
  * Each pointer is a "fat pointer", consisting of the offset from the start of the page, and the length of the tuple.
* This will also work for fixed-length records to enable compacting.

### System Catalog - How to Find Pages and Their Formats

For each table, the System Catalog has the name of the table and the header block of the file, the structure of the file (heap file, B+ tree file), the names, order, and data types of each column, references to each index (and its type) on the file, and any constraints (like uniqueness). Their are also views on the table stored here -- with the view name and the definition. Finally there are statistics, authorization, buffer pool size, and other metadata.

Note that the Catalog is itself a table, so it itself is in the Catalog! So the Catalog table is stored on the disk in a pre-defined file in a pre-defined format.

There may be multiple catalog tables, depending on the complexity of the DBMS.

## Buffer Management

We want to be able cache and pre-fetch from the Disk into RAM so that query processing goes faster. The buffer manager will hide the fact that not all of the data is in RAM at one time. This is very much like how the OS hides the disk.

In main memory, a pool of buffer frames, each the size of a disk page, will be allocated, with a map that associates Page IDs with particular frame numbers, and whether the page is dirty. If a page is requested and is in the pool, it will be returned without going to disk. If a page is not in the pool, then we will have to execute the cache replacement policy (at minimum, if the page is dirty, it needs to be written to the database before replacement).

Pre-fetching can also be done, in the case of sequential scans, to put these pages in the buffer pool, in some background thread.

### Replacement Policies

In all policies, "pinned" pages (pin count > 0) cannot be replaced. What is pinning? It is a count of how many callers need this particular version of the page. The caller must eventually unpin the page (decrement the pin count), and then mark the dirty bit as necessary.

Concurrency control and recovery may also do some additional IOs during replacement, like journaling, to enable recovery and correct concurrency. The OS versions of these may not be correct, so the DMBS may need its own.

The efficacy of replacement policies often depends on the workload. But with DBMS, since we know the query, we will know the workload! Does this allow us to optimize more than in the OS? Replacement policies (heuristics):

* Least Recently Used (LRU)
  * Ignore the pinned frames
  * When pin count goes to 0, time-stamp the frame with its last time of use.
  * Replace the frame with the oldest timestamp.
  * Works well for hot-cold workloads, with a lot of temporal locality.
  * Doesn't work well in sequential flooding, where each page of a file is requested in sequence, multiple times.
    * Repeated sequential scans are a problem if there are fewer buffer frames than there are pages in the file. In that case, you will have a buffer miss with every single access.

















