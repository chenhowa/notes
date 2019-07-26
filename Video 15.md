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

### System R (Selinger) Style Optimization

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
  * Therefore we can do joins in any order.
    * R x (S nat.join T) = (R x S) nat.join T

45:00
