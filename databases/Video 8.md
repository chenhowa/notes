# Relational Algebra

## Aside: B+ Tree Comments

In Alternative 1, the leaf pages contain the actual data records, and splitting changes the record ids of multiple records, so all pointers to them have to be updated. This is expensive, and the updates typically don't have cache locality, making it even worse -- lots of IO's.

But in Alternative 2 and 3, the leaf pages of the B+ Tree contain only pointers to the actual records, so splitting does not change the record ids of all the data records.

B+ Trees are great for a large number of small lookups. But in analytical lookups, where all the data is grabbed, you would just use a sequential scan of the heap file. Since B+ Trees are the workhorses of the DBMS, they are heavily optimized and tuned for specific use cases.

Main-memory databases also have highly-modified B+ Trees, since the IO patterns are very different (since disk IO is much less likely).

## Relational Query Languages

These are for manipulation and retrieval of data, founded on first-order logic, while allowing much optimization. Since it is a "relational" language, parallelization is much easier to implement than in "sequential" languages.

These query languages allow for a small DSL for  data that allows for rich program analysis (and optimization). *This is why it's useful, in addition to data lineage (where did the data come from, and why?), materialized views (query whose output has been saved), updatable views (query whose output can be updated, leading to an output in the data itself)*.

Query languages typically aren't Turing-complete, since they're just small DSLs for data processing. They aren't intended for complex calculations. But since big data now requires large calculations, query languages are being extended to do complex calculations in parallel, and becoming well-suited to the tasks.

Perhaps query languages will become appropriate for other similar data tasks where the data can be modeled as a stream, where this query language is more appropriate.

* The Relational Algebra
  * Operational, representing execution plan semantics
* The Relational Calculus
  * A declarative language that describes *what* you want, but not *how* to compute it. This is the foundation for SQL.

Relational Algebra and Relational Calculus have the same expressivity, which means that a statement in relational calculus (SQL) can be compiled to relational algebra (operators, iterators).

An input to a query is an instance (table). The result is also an instance (another table). Hence the query language is closed, making them composable.

The algebra does not allow duplicates in relations

* Selection - get rows that satisfy some selection condition. Does not affect number of rows.
* Projection - gets desired columns
  * Eliminates duplicates
* Cross Product - combine relations - all pairs from the two input relations. Field names are "inherited" when possible, from the inputs. If the two inputs have two columns with the same name, you can use a renaming operator to resolve the conflict.
* Difference and Union - set operations over relations. Requires that the relations have the same schema (same number of fields, and corresponding fields have the same type)
  * Union - output has all the rows, minus duplicates
  * Difference - output has all things in S1 that are not in S2.

Why not eliminate duplicates?

* It might be useful to know the expected output size (which is the input size)
* It's expensive

### A Note on Set Difference

Note that all operators except set difference are "monotonic" -- as the input grows, the output grows.

But Set Difference is not. The larger the relation in the right-hand input, the smaller the output is likely to be (it certainly won't be bigger). This has implications

This means that the Set Difference is a *blocking iterator*. You cannot begin computing the output of S - R until you've seen all of R (think about it).

All other operators can be implemented as *non-blocking iterators*

### Intersect

We can define intersect in terms of the other operators: R - (R - S)

### Natural Join

Compute the cross product, and select the rows where some attributes in both relations have equal values, and keep only the columns that are unique to each relation, renaming as necessary (as well as one copy of the equality columns)

Of course, in an actual implementation, we should not calculate the full cross product.

### Theta Join

Filter the cross product tuples according to a theta function. No projections are done unless you ask for them. Equi-Joins mean that the theta function comparing columns from both tables only involve equality. Theta functions should use Condition AND Condition, not Condition OR Condition. Because AND allows for just one query with indexes, while OR requires multiple queries (it's essentially a union of sub-queries)

### Efficient Solutions

To make more efficient algebras, throw away as many columns as possible before doing joins. Having skinny rows means that there are more tuples per page, reducing the number of IO's by a lot.

Column-oriented databases take this idea to the extreme -- the columns of a relation are stored separately on disk, so projection is done before scans. Very weird, but it enables certain optimizations.

### Associativity and Commutativity

What are the rules? We'll see them in a later lecture.










