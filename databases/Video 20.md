# Big Data Analytic Systems

In databases traditionally, there is a split between transaction processing (OLTP, doing lookups and doing updates), and analytics (OLAP, compute metrics and KPI's to drive business decisions). Typically, analytics involves scanning and manipulating a huge volume of data compared to transaction processing.

Interest in Big Data took off over the last two decades. Gartner's definition of Big Data is "high-volume, -velocity, and -variety ...". 

* volume - data size
* velocity - how fast the data is changing
* variety - many kinds of data sources, formats, and workloads (methods you use to analyze the data)

Big Data can also refer to the tech stack that is used for working with data at the scale of Google, these days. Because data is high-volume and high-velocity, to process it with high-performance, you need to spread out the workload over many clusters of machines.

## The Big Data Problem

Semi/Un-structured data often doesn't fit well with databaess. A single machine cannot process all the data. The only solution is to distribute storage and processing over clusters. Hence, in 2003, Google published a paper about the "Google Filesystem"

* Assumptions
  * Component failures are the norm, not exceptions, due to Google's scale.
  * Files are way huger than traditional standards.
  * Most files are more immutable -- they are mutated through appends, not updates.
* If you have a large file, store it as a bunch of equally sized blocks, over many different machines. Each block may be replicated over several machines (say, duplicated over 3 machines)
  * First copy is one the node creating the file
  * Second copy is written to a node on the same server rack (no need for much network traffic)
  * Third copy is written to a node on a *different rack*, to tolerate failures if the whole rack fails to be connected to the network.
* To get load balancing, fast access, and fault tolerance
* So there are two kinds of nodes
  * Data nodes store data
  * Name nodes for, heartbeat, coordinating data nodes and tracking the metadata of blocks. There may be backups for the Name nodes

### Handling Failures

* Disk errors/failures
* DataNode failures
* Switch/Rack failures
* NameNode failures
* Datacenter failures

### Distributed Programming

Traditionally, it was done with Message Passing or Remote Procedural Calls. This didn't work well for the GFS

* How to split the programming across nodes?
* How to deal with failures?
* What if a node is slow?

Instead, GFS popularized Data-Parallel Models, taht restrict the programming interface so taht the system automates more. "Here's an operator,  run it on all of the data. You schedule the everything -- on which nodes, in what order, etc.". This is very similar to SQL, and letting the query optimizer handle *how* things are done.

Google came up with *MapReduce* for this purpose, in 2004, which was popularized by Hadoop. It was the first popular programming model for data-intensive applications. The MapREduce engine automatically splits work into many small tasks based on data locality, and a dynamic load-balancing considerations. For fault recovery, if a task fails, just re-run it (re-fetch its input if necessary). If the failures exceeds the maximum number of failures, tell the user that it failed. If a node tails, rerun its tasks on other nodes -- but this requires that the task result is independent of the node it is run on, and it has no side effects from being run multiple times (for example, you can't append twice and have the result be correct). Also, if a task is slow, run it again on another node (maybe this node will finish faster).

*All this happens automatically after the programmer declares what they want!*. Also, MapReduce is much more powerful than SQL -- it can be used to crawl the web lol.

Hadoop was open-sourced by Yahoo, and built based on the two Google papers.

### Criticisms of MapReduce

MapReduce could be considered a strict subset of DBMS that is implemented *worse*. When this criticism was made in 2008, this started a war.

So why didn't Google user Databases?

* Cost - databases vendors charge by TB, and Google was working with even more data.
* Scale - no DBMS, at that time, had been shown to work at that scale (amount of data, number of nodes)
* Data Model - a lot of semi/un-structured data in web pages (images, videos, etc.). At the time, DBMS's did not really support such data.
* Compute Model - SQL was not expressive enough for Google tasks like crawling the web and building inverted indexes.
* Not-invented-here - Google just wanted to build its own stuff!

Is MapReduce less programmable?

* Most real jobs require multiple chained map-reduces.
  * A google indexing pipeline was 21 MR steps
* Multi-step jobs create spaghetti code with tons of boiler plate and multiple classes

Higher level frameworks built on MapReduce were therefore built, like Hive (SQL to MapReduce) and Pig (SQL-like query language to MapReduce). Now, 90% of MR jobs are generated by Hive SQL.

Another problem with MapReduce is performance. Each MR job writes all output to disk, instead of streaming to the next step, or broadcasting data to multiple nodes. This means you have a lot of IO's, and even more due to replication.

## Apache Spark

Designed to dress problems with MR. Programmability follows the model of applyling functional transformations to collections, and much less boilerplate. Spark programs can even be unit-tested. Performance is also better -- 10x to 100x faster. Multi-stage MR is used, allowing for streaming of data from one operation to another, without writing completely to disk. Richer primitives are available (like broadcasting and in-memory caches)

Now there are SQL engines on top of Spark, as well as frameworks for machine learning, graph computation, and R

As the audience for Spark has widened, the need for data schemas has arisen, leading to "DataFrames" in Spark. It is a DSL that lets you work with data that is grouped into named columns, making it easier to work with.

## The Return of SQL

Despite NoSQL, SQL has been making a comeback. Having solid schemas of some kind always ends up being important for users.

* Almost everyone knows SQL
* Easier to write for analytic queries than both MR and Spark
* Schemas are more useful than the (key-value) model of MapReduce


### What's the difference between SQL platforms?

Both BD (Hadoop/Spark) and DB use SQL, so what's the difference?

* Flexibility in data and compute model
  * In DB, the data must be structured, and the query goes in, and the output comes out.
  * In BD (Big Data), storage is decoupled from the data engine (you can add data without talking to the compute engine). There are multiple ways to use the engine, both low-level (MR) and high-level (SQL, DataFrame). Structured and unstructured data are bo4th supported, and schemas can be used for both read and write
* Fault-tolerance
  * In DB, there is course-grained fault tolerance -- if there's a failure, the query fails and must be restarted.
  * In BD, the fault tolerance is fine-grained. Individual failed tasks are run, not the entire query.

*Most organizations don't have queries that require BD systems*. They don't need that fault tolerance and flexibility.

In Hadoop, intermediate results are always written to disk as a *checkpoint* to ensure fault tolerance. In Apache Spark, you don't checkpoint unless failures start becoming frequent, so for most workloads (the small ones), you don't pay the cost of checkpointing, but you still complete the query.



