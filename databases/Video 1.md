# Introduction to Databases

## The Why of Studying Databases

Data is at the center of most technology today.
The amount of data being stored now is huge.
Most of the data is copies of other things, like movies.
Much of the data is generated through automation (software logs, cameras, microphones, RFID, GPS, etc.). Yup.

## The What of Databases

Many websites utilize multiple databases on the server to deliver content like surveys and recommendations. Many devices that can gather data (phones, smart watches) are often backed by databases. *Databases are large collections of structured data*

Hardware and Data Volume are changing the assumptions behind DBMS technology. Some classic algorithms cannot be used in certain configurations.

### Is an OS a DBMS? Good Question

OS's have file systems, and data can be stored in RAM. Files can be protected, and they are stored on persistent disks. Although files are slower than RAM, by an order of magnitude. Files are not random access (assuming magnetic disk), and the API for accessing particular indexes are not as good as RAM pointer indexing. Although both File System and RAM have concurrency issues.

File systems don't necessarily handle concurrent writes or crashes in a way that is appropriate for databases. Especially because the OS may have just been buffering writes without journaling the entire write first.

*Good* DMBS systems provide abstractions that are geared towards handling these issues more clearly, with regards to concurrency, replication, and recovery. The guarantees are much stronger. The APIs for working with DBMS should be a good DSL (query language) that runs on efficient, scalable bulk processing, with good data modeling, so at the application level, programming is easier.

### Big Data, Big Companies

Many large companies developed DBMS's that worked well for them, but not anyone else. When thinking about scale, most systems don't need to scale to the largest companies, since their scale leads to unique problems for performance that most companies won't have to deal with.

Variants: In-memory databases (main-memory databases), and row-oriented databases vs column-oriented databases. The current market is very interesting, but it is all based on the foundational CS. NoSQL - MapReduce, Spark, Cassandra, MongoDB. Text Search products often require databases as well. Cloud services also need to provide database services.

## Basics of Big Data

Von Neumann machine -- instructions are stored outside the CPU, and then loaded into the CPU to be executed. In today's CS world, there are multiple machines, with one storage abstraction per machine. You could have one giant storage for all the machines (a la Hadoop), but the more common case is that data is spread out across lots of computers.

## Basic Patterns for Big Data

There are two: Streaming and Divide and Conquer.

Assume that you only have unordered collections of data items (multi?-sets), and that you can handle them in an unordered fashion. *These assumptions are the basis of scalable systems, because now cache locality and rendezvous and arbitrary batch sizes can be organized*. Parallelism now becomes easier to do, as long as everything is unordered.

### Streaming

How do you stream data through RAM while using just a small amount of RAM and not getting performance penalties from I/O? So we want to amortize the costs to RAM and from I/O.

One way to do this is through streaming. Stream the file through memory, a few data bytes at a time. This data will fill an input buffer. Items will be taken out of the input buffer, processed, and then placed in an output buffer. Once the output buffer is full, write it (incur the I/O cost). When the input buffer is empty, read it to full (incur the I/O cost). Observe that the input filling and output writing may not be occurring in sync, based on the size of the processed result.

Streaming can be easily parallelized, and the results can be merged later. You just have multiple machines with their own input and output buffers.

If the data was ordered, the parallelization would not work.

### Rendezvous

Make sure that two items are in memory at the same time (same space at the same time). Rendezvous is required for certain algorithms, especially if the data are coming from different Streams, for example.

### Divide and Conquer

Divide and Conquer often uses rendezvous, where we first divide up the data into groups that need to rendezvous together. These algorithms are often called "out-of-core" algorithms, which means "out-of-memory", since performing these algorithms often requires the use of disc memory to store the groups, since the data is way bigger than RAM.

For such algorithms, if you have B chunks of RAM, use 1 chunk for input, 1 chunk for output, and (B - 2) chunks for rendezvous and processing of groups of data.

A typical example is to first use a stream algorithm to divide the data into N / (B-2) groups that are written to disk (may process in this stage). Then, you use a streaming algorithm over each of the partitions, with rendezvous if needed, to process each partition of the data as a group.

How can the division be parallelized? Especially considering that data that belongs to a partition may be spread across multiple machines? What we can do, is to then send all the data in a partition to a single machine first, for storage of an entire partition, before performing the second processing phase.


