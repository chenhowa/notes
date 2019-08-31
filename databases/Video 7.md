# Tree-Structured Indexes

Indexes can be based on several on-disk structures. Here we will discuss only indexes based on B+ Trees.

Recall that record ids are (page #, slot#) tuples. Variable length data means that the offsets of each field of a record are stored in the record's header, and the offsets of each record are stored in the page.

Since there is overhead to free space management and exact ordering, these will be cleaned up "lazily" in batches.

Indexes ("access paths") are more flexible and less expensive than sorted files.

## Basics

Selections are done in the form field-op-constant. Equality and range operators are available. Tree indexes work for both; hash indexes only work for equality.

ISAM (Indexed Sequential Access Method) is an early indexing data structure that is interesting to look at. B+ Trees are the goal, however.

## Range Searches

If we want to find all students with GPA > 3.0, what can we do? In a sorted file, we would do a binary search, and then a linear scan, but *random disk access* is expensive. Instead, we can build and maintain an index file on top of the sorted file. The data entries of the index will store, at minimum (k, page pointer) data entries that make it easy to go find the pages that contain the range you are interested in. But the index itself may be quite large.

### ISAM

So in ISAM, you have a sorted file, and a hierarchy of index pages is built on top of this sorted file. Each index page contains (k, disk pointer), that says that if you follow the pointer, you will find all pages with values >= k (unless you use the negative infinity pointer). As you go down the hierarchy, the ranges are carved into smaller and smaller chunks, until you reach a pointer to the pages that have the range you want. Note that we don't need "next-leaf-page" pointers in the leaf pages (the data pages), because we know the data pages are sequential and sorted, starting from the front of the file.

To create an ISAM (a STATIC structure), you do the following:

* Sort your file. These are the leaf pages; the data pages.
* Construct the index pages, and append them to the end of the file.
* When needed, create overflow pages.

Searching starts at the root of the index, and uses key comparisons to find the leaf where a range starts. Then the cost to find the leaf is log(r)(N), where r is the average number of children per node.

Inserting means finding the leaf data page, and putting the data in there. If it is flow, *overflow the page*. That is, allocate an overflow page at the end of the file, put a pointer to the overflow page in the leaf data page. Since it is not sorted, we will now have to sort the leaf and its overflow pages in memory every time now. Overflow pages will then be chained with disk pointers, as needed. *So we see that ISAM degrades as the number of insertions grow*, and the ISAM will need to be rebuilt to restore performance.

Deletion - Find the record(s), and delete it from the page. If it is the last on an overflow page, we just de-allocate the overflow page. No that on deletions, there is no need to change the index pages, since those just act as range dividers for navigation.

Pros of ISAM

* Data is sequential! That's pretty great (especially if, for some reason, the data is write-never)

Cons of ISAM

* ??

### B+ Tree: the Most Widely Used Index

Like the ISAM, it will be a height-balanced tree. Insert/Delete is log(f)(N). B+ is not width-balanced -- different nodes have different fan-outs. However, each node will have minimum 50% occupancy, besides the root (that is, between d and 2d children). "d" is called the *order* of the tree. Range searches go from the root to the leaves, but the structure is *dynamic*, unlike the ISAM, which is *static*. Also, B+ Trees requires pointers from leaf to leaf (doubly-linked) (data page to data page), since the data is no longer sequential.

Once we have a page, we can do an in-memory binary search in the page to find the desired value (if it's present ). The initial top levels of the index can typically be held in the buffer pool, reducing the number of IO's to get to the leaf level.

To do an insertion,

* Do a search for the correct leaf, and if there's space, put the information in there,
* If the leaf is full, we split the leaf node in half, put 8 in in the correct sorted order, and try to put a new data entry in the index page above.
* If the index page is full, we split the index page into two, add the new data entry in the new space, and then add a new data entry in the index page above.
* If there is no index page above, we create a new root index page, *push up* the middle data entry in the new root, making the B+ tree taller.
* You can see that this happens recursively while keeping the B+ tree height-balanced.

To do a deletion, naively:

* Do a search for the correct leaf, and remove the entry
* If the leaf is still at least 50% full, you're done.
* If the leaf is less than 50% full, try to borrow data from a sibling node, and then adjust keys in the parent index node
* If borrowing fails because the sibling has too few entries, merge the leaf and the sibling, and adjust the data entries in the parent (this involves a removal from the parent).
* This might propagate up the tree to the root, shrinking the tree (once the root only has 1 child, it is no longer needed).

In practice -- just let the nodes be less than 50% full. This is okay because typically there aren't many deletes in a database, and even deletes are followed by inserts. So typically the above deletion algorithm isn't implemented.

### B+ Tree Tips

Fan-out should be made as large as possible, because it makes the tree shorter, reducing the number of IO's.

We can do compression of data entries to get more entries per page.

* Prefix Key Compression
  * Take the prefix of the key. As long as the traffic is still directed correctly, there is no problem. For example, if the search key is "full name", you might just use the first name to direct traffic with more data entries per page.
  * The Compression often happens just when the insertion splitting happens.
* Suffix Key Compression
  * If many index entries have a common prefix, we might direct by the suffix only.
  * This is particularly useful for composite keys, if the first field you index on has many duplicates.

So if you have a heap file, you can build a B+ Tree index on the heap file. The data pages will contain record ids, sorted according to the record key, even though the heap file itself is not sorted.

### Bulk Loading of a B+ Tree

How do we bulk load a B+ tree? If we have a large collection of records, and we want a B+ tree on some field.

But running insert over and over on the B+ tree, from a heap file, is a bad idea: It's slow (because lots of random IO's per tuple of insertion).

So to do this, we sort the file first (into a new file), then insert a pointer to the first leaf page in a new root page, and then insert in the order of the sorted file, gradually building the structure of the B+ tree over it. When doing this, we can control how full the leaves are as well, by not filling the sorted file's pages completely.

Since we are always inserting to the right of the tree, we get better cache hit rates if we are using LRU.

### Order

"Order" makes less sense with variable-length entries, since the number of children depends on how big the variable-length tuples are. So we often only talk about how full, percentage wise, a leaf or index node is. And many systems let nodes be less than half full anyways.

B+ Trees are almost always better than maintaining a sorted file.

Note that if the leaf pages are actually in the file, then splitting entries may actually change the rid!

So, two options when building an index -- copy and sort the entire file, and build the index on top of those leaf pages; or -- build an index where the leaf pages are record ids. However, this second option seems potentially expensive to build.








