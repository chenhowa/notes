# More on Query Optimization

Recall that:

* The considered plan space is left-deep trees.
  * Joins are commutative and associative (although the implicit selection operator might have to be pulled out)
* There are various access methods for the leaves of the trees:
  * B+ trees, heap scan
* Various join algorithms 
  * Chunk-nested loops
  * Sort-merge join
  * index-nested loops 
* Selection and projection operators can be pushed down in many cases 
  * Therefore we try to put these operators as low in the tree as possible

Total cost is estimated from the cost of inputs, cost of outputs, size of outputs. This is done through selectivity estimation, which is based on information stored in the catalogs, and formulas


## Selectivity Estimation

Consider ther query "SELECT * from sailors WHERE rating > 5". How do we estimate how many sailors will be output from the query? Naively, we assume uniform distribution of sailors across the range of rating values.

But we can do better if 

  * in the catalog, we store parameterized statistics that describe the distribution (mean, median, variance, standard deviation). But this is tough, because it requires fitting the data to a curve, which can be an expensive calculation.
  * in the catalog, we non-parameterized statistics that describe the distribution (like a histogram that stores the number of values in small ranges of the data).
    * The ranges in the histogram don't have to be "equi-width" (they don't all have to be the same"). We can also use "equi-depth" histograms that divide the data into ranges, where each range contains the same count of data (for example, 10% of the data)
    * Equi-depth let you look at medians and quantiles.
    * Equi-width is sensitive to outliers in the data.
    * There are trade-offs between the two.
  * Can also try sampling the data to get a sense of the distribution, and then get parameterized statistics from that

Now consider the join query "SELECT * from sailors S, reserves R WHERE S.rating > 5 AND R.sid = S.sid". Since we're joining two tables, we have two histograms. How do we combine the two to do selectivity estimation on the join, if we're using *histograms*?

We take the overlapping ranges of the two histograms, and calculate the probability that a join will be found within each range (use the col1 = col2 reduction factor calculation from the previous lecture). Take the average reduction factor. *Note that this works for equi-width histograms much better than equi-depth histograms*.

## Search Algorithm (Selinger System R Algorithm)

Two main cases, inductively:

* Single-relation plans (base case)
*  Multiple relation plans (inductive)

### Single-Table Query

Chooset the access path with the least cost. Order if necessary, and then do as much selection/projection as possible, and then pipeline into grouping/aggregation.

Costs for plans:

* If there is an Index I on the primary key that matches selection, the cost is Height (I) + 1 for a B+ Tree, since you walk the tree once to find the single record that matches
* Clustered Index I that matches one or more of the selects
  * Cost is (NPages(I) + NPages(R)) * product of RFs of matching selects, since the pages in I correspond to the pages in R, and you walk down both in order.
* Non-clustered index I matching one or more selects 
  * (NPages(I) + NTuples(R)) * product of RFs of matching selects, since you will go down the pages of I, but you will have to do random-access on the tuple sin the heap file. You are likely to get one IO per tuple
  * You may have to estimate NTuples(R) by estimating the distribution of the data, as discussed earlier.
* Sequential scan of heap file has a cost of NPages(R)
* If duplicate elimination, or actual sorting of the data is required, then you pay for the Sort/Hash as well

### Multiple-Relation Queries

To enumerate all the Left-Deep Plans

* Vary the order of relations, the access methods for the leaves (relations), and the join methods for each join.
* Pass 1 - find best 1-relation plan for each relation
* Pass 2 - find best way to join result of the (i-1) relation plan to the i<sub>th</sub> relation.
* For each subset of relations, we keep only the cheapest plan overall, and the cheapest plan for each *interesting order* of the tuples. Interesting order means the result has been sorted by "ORDER BY" attributes, "GROUP BY" attributes, or JOIN attributes that will be required for a later join.

The dynamic programming table will have 4 columns -

* the subset of tables being considered for the row, 
* the interesting order columns (if any for this plan), 
* the best plan, 
* and the cost of this plan. We keep track of interesting order columns so that we can use it later -- it might be cheaper to sort earlier, rather than sort later, for a later operation

Now, when do we match an (i-1) way plan with the i<sub>th</sub> relation? We do it if there is a JOIN condition (possibly in a WHERE clause), or if there is no join condition (but now we have to do a cartesian product, which sucks). We do cartesian products as late as possible

To handle ORDER BY, GROUP BY, and Aggregates, we'll consider them at the final step, and either calculate them with an "interestingly ordered" plan that we've found so far, or through one final sort/hash (whichever costs less)

Unfortunately, despite this pruning, this is *exponential in the # of tables*, since a enumerating all left-deep trees is very similar to enumerating all possible strings from a given alphabet.

Recall: Cost is estimated as #IOs + (reduction factor * #tuples)

### Physical DB Design

The database admin typically decides what indexes there are (since building indexes is expensive in both disk space and performance). Therefore DBAs understand query optimizers very well, since they need to know which queries are improved when they choose to add an index. The DBA needs to decide if it's worth it, and if it will improve a query at all.

* Which tables shoudl have indexes?
* What fields shoudl be the search key?
* Multiple indexes?
* Clustered or unclustered?


The greedy approach is to consider the most important (frequent?) queries in turn, and see if a better plan is possible with an additional index, and whether adding that index takes too much disk space or slows updates too much (you have a budget).

#### What Indexes Should You Build? Good Question

* Attributes mentioned in the WHERE clause of queries, or in JOIN clauses. Remember that Range conditions are much better with clustered indexes, than with unclustered. Clustering will also help Equality conditions if the column(s) is not a key.
* Choose indexes that benefit many queries.
* Remember that only one index can be clustered per relation, since a file can only be sorted in one way! So choose it wisely.

Some systems will log all queries they ever see, and use that log to help recommend queries to DBAs through a "tuning wizard" software.







