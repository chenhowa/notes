# Relational Query Optimization

In optimization, what can we optimize?

* Choice of operations (indexes, memory, stats)
* Joins (rules of thumb)
  * chunk-nested loops - simple, uses all extra memory
  * indexed nested loops - best if 1 rel is small and the other is indexed 
  * sort/merge join - good with small amount of memory
  * hash join - fast, bad with skewed data

## Query Optimization Overview

Queries can be converted directly to relation algebra, which is converted to a tree. Each operator has an implementation choice; and the operators can be applied in different orders.

A plan is a tree of operators, where each operator has a choice of algorithm. There are three main issues:

* What plans are generated from a query?
* How do we estimate the cost of a plan?
* How do we find good plans efficiently?

Queries are parsed, and then the query optimizer generates plans from the tree and estimates the costs. The Cost Estimator looks at the table schema and statistics, and uses it to estimate costs. The Query Parser may rewrite the tree (for example, turn subqueries into joins), to improve the optimizations that the Query Optimizer can find. Once a Query Plan is chosen, it is executed.

Example optimizations (without indexes):

* Push select operators down below joins in the tree, so that total number of tuples is reduced early. 
* Cache the results of operators by writing them to temporary files. If the temp files are smaller in the first place, this caching can be very helpful.
* Can push projection operators down to reduce size of tuples. This usually doesn't make a difference if you aren't writing the output down to disk (if it's only streaming through memory)

With indexes, these optimizations often don't lead to improvements.

## Ingredients for Optimization

Given a closed set of operators, we need 

* An enumeration of plans (based on relational equivalencies on the relational algebra)
* A way to estimate cost based on
  * Cost formulas
  * Size estimation, based on 
    * Catalog Info from the base tables
    * Selectivity (estimating how much pushing down the select operator reducing the total tuple count)
* Search algorithm - how to sift through the plan space to find the cheapest option.

## System R (Selinger) Style Optimization

Works well for up to 10-15 joins. It approaches the ingredients as follows:

* Plan Space - Too large. It must be proved!
  * Many plans share common, "overpriced" subtrees. Ignore all plans that have the overpriced subtree (a heuristic)
  * Maybe only consider "left-deep" plans
  * Maybe ignore Cartesian Products
* Cost Estimation
  * Can be inexact, still works okay in practice (any plan that is 100 times better is good enough)
  * Stats in system catalogs used to estimate sizes and costs 
  * Consider combinination of CPU and I/O costs
* Search Algorithm - *Dynamic Programming*
  * Memoize the highly expensive subtrees when generating and deciding on candidate plans.

### Examine the Plane Space

#### Query Blocks: the Unit of Optimization

Break the query into query blocks, and optimize one block at a time. Uncorrelated nested blocks (blocks that don't reference other blocks) only need to be computed once, and cached. 

Correlated nested blocks need to be computed multiple times based on the input of the other referenced blocks (kind of like a function call). A block could be "decorrelated" -- an advanced topic.

For each block, the plans considered are 

* All available access methods for the relations in the FROM clause (heap, index, sorted files, etc.)
* All left-deep join trees. We won't consider trees with any other structure.
* Consider all join orders and join methods

A left-deep join tree is a binary tree such that the right-branch is always a base table, and it looks like this:

JOIN( 
    JOIN( 
        JOIN(
            table, 
            table
            ), 
        table), 
    table
    )

#### Relational Algebra Equivalences

The following operations maintain equivalency

* Selections
  * Cascade - push selections down the tree
  * Commute - can switch the order of successive selections
* Projections 
  * Cascade - projections can be done earlier, as long as the final projection is still done at some point.
* Cartesian Product
  * Associate - R x (S x T) = (R x S) x T
  * Commute - R x S = S x R
  * Therefore we can do almost do joins in any order, since joins are just cross products with selects built in.
    * R x (S nat.join T) = (R x S) nat.join T
      * This may not be a good thing. (R x S) may be way huger than (S nat.join T), which leads to much more expensive IO cost.
    * But be careful!
      * ( R nat.join S ) nat.join T != (R nat.join T) nat.join S, since R and T may have no columns in common. So instead we have ( R nat.join S ) nat.join T == *select*((R x T) nat.join S), which is not what we want -- the cross product is way bigger and more expensive. The result is the same, but we had to move the *select* up and out.
* Eager projection 
  * project<sub>i</sub>(select<sub>a=4</sub>) = project<sub>i</sub>(select<sub>a=4</sub>(project<sub>i,a</sub>))
  * Selection on a cross-product is equivalent to a join
  * A selection on attributes of R commutes with (R x S)
  * But overall, these are situational. You have to make sure the information you want is always available when you make these situational equivalencies.

#### Queries Over Multiple Relations

The System R heuristic -- only consider left-deep join trees. Why?

* Restrict the search space.
* Left-deep trees allow the *generation* of all *fully pipelined plans*
  * That is, intermediate results do not have to be written to temporary files -- streaming can be done very easily, reducing the disk footprint. Not all left-deep trees are fully pipelined, however.

### Examine Cost Estimation

Estimating the total cost of a plan 

* Estimate cost of each operation
  * Depends on size of inputs
* Estimate *size of result*, since it determines size of inputs to downstream operations.
  * Estimate based the input relations.
  * For selections and joins, you will assume that the predicates are independent, as a heuristic.
* In System R, the cost is aggregated into a single number consisting of #IO's + (CpuTimeFactor * #tuples). This is sloppy (what is CpuTimeFactor? The assumptions are not that accurate), but it works in most cases as a heuristic for ordering the plans. *You avoid the worst plans, and pick a pretty good plan (not necessarily the best).

#### Using the Catalog and Statistics

In the first System R, catalogs typically stored:

* number of tuples in a table.
* number of disk pages in a table
* min/max value of each value in a column (High(col), Low(col))
* number of distinct values in a column (NKeys(col))
* height of each index on the table
* number of disk pages in an index

Catalogs are only updated periodically (too expensive to do on-the-fly). The approximation is usually okay. Modern systems keep even more data.

#### Size Estimation

Size estimation can be done with a number of heuristics:

* Max output cardinality = cardinality of cross-product of the inputs
* Selectivity - we are throwing away tuples with each selection. How many are being thrown away?
  * We estimate the selectivity of each term in the Select clause (how much does "age > 5" reduce the output?)
  * |estimated output| / |input|
  * Multiply the selectivities of all terms together to get the final "Reduction Factor"
* Result cardinality = (max # tuples) * (composite Reduction factor).

Examples Reduction Factor calculations:

* col = value (given number of distinct values NKeys(col) in the column)
  * RF ~ 1 / NKeys(col) (assumes uniform distribution)
* col1 = col2         (handy for joins as well)
  * RF = 1 / Max( NKeys(col1), NKeys(col2) )
  * This factor is based on probability of finding two records where the two column values match.
* col > value
  * RF = (High(col) - value) / (High(col) - Low(col) + 1)
  * This factor is another statistical calculation: (range size of query) / (range size of table). Huge assumptions are made here about uniformity.
* According to SystemR designer's advice, choose the RF as (1/10) if the relevant Catalog statistics aren't available. This value is arbitrary. You may want to choose one for each operator that doesn't have the statistics.







