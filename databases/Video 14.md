# Schema Refinement - Normal Forms

## Boyce-Codd Normal Form (BCNF)

Recall - the only dependencies are on key columns, superkey column sets, and trivial dependencies (superset of columns determines a subset of those columns)

Then there is no redundancy.

## Decomposition

To normalize a relation, decompose the relation into multiple normalized relation.

Suppose R has attributes A<sub>1</sub> to A<sub>n</sub>. Then the decomposition consists of replacing R by two or more relations such that 

* Each new relation scheme contains a subset of the attributes of R, and
* Every attribute of R appears as an attribute of at least one of the new relations.

To decompose for normalization, decompose the relation into a main table that has references to other relations that contain the functional dependencies. All the resulting relations should be in BCNF form.

### Problems with Decomposing Too Much

If you decompose too much, then all your queries will have to be joined back together (performance and code-writing cost). If you decompose incorrectly, you *can't* join the relations back together (you've lost data associations)

If you decompose SLRH into SL, SR, and SH, you do get a benefit -- if you mostly look at SL, but not at SR and SH, then caching and disk scan performance is improved. because SL is less data (the cache is affected less, and the disk scan scans less). This is called *vertical partitioning*, and when this is taken to the extreme, the industry calls this *column store* technology. But if you are doing a ton of joins, *vertical partitioning* can be a loss.

* Lossiness - may be impossible to reconstruct the original relation. To avoid this, make sure that the foreign keys are done properly in the resulting relation (the common columns need to b compositely unique)
* Dependency checking may require joins. Dependency checking is a kind of integrity constrain; for example, if A -> B and C -> B, and ABC is decomposed into AC and BC, checking that A -> B now requires a join.
* Some queries are more expensive due to joins.

The decomposition of R into X and Y is lossless with respect to the FD set F *if and only if* F+ contains

* X intersect Y -> X, OR (the shared columns determine all of X)
* X intersect Y -> Y (the shared columns determine all of Y)

Useful result: if W -> Z holds over R, and W intersect Z is empty, then the decomposition of R into (R - Z) and WZ is definitely lossless.

### Dependency Preserving Decomposition

How do we decompose without requiring that we do joins for dependency checking?

If you decompose R into X, Y, and we enforce all the FDs in F on each of X, Y, and Z, then all FDs on R are also enforced without any additional work.

Projection of set of FDs F: If R is decomposed into X and Y, the projection of F on X (F<sub>x</sub>) is the set of FDs U -> V in F+ such that all of the attributes U, V are in X.

The decomposition of R into X and Y is *dependency preserving* if (F<sub>x</sub> union F<sub>y</sub>)+ = F+. This basically says what we said above: if we split R into X and Y, the functional dependencies on each independent table end up being the functional dependencies on R.

How do we test whether a decomposition is dependency-preserving?

### Decomposition into BCNF (Dependency-Preserving?)

Consider relation R with FDs F, where X -> Y violates BCNF. Then decompose R into (R - Y) and XY (guaranteed to be lossless). Several dependencies may violate BCNF. The order in which we decompose could lead to very different sets of relations.

*Unfortunately, there may not be a dependency-preserving composition into BCNF!* Example: 

* CSZ, CS -> Z, Z -> C
  * Try splitting it into SZ and ZC
* This cannot be decomposed while preserving the first FD.
* This typically occurs when decomposition splits apart 1-way transitive functional dependencies; in this case, CS -> C, as in the example above.

One way to solve this is to have some of the dependencies be stored redundantly in their own tables, avoiding the join (code is needed to enforce the dependency, instead of structure). This may not lead to all tables being in BCNF, unfortunately. In this way, if you have an update X that needs to be put into the two tables X and Y (due to redundancy), but the only purpose of X is to check the dependencies, then you only update Y if the X update succeeds.

## Third Normal Form

This is the solution to the fact that BCNF cannot necessarily be decomposed while preserving dependency.

A relation R with FD's F is in 3NF if, for all X -> A in F+:

* A is in X (trivial) (also a rule of BCNF)
* X is a superkey of R, or  (also a rule of BCNF)
* A is *part* of some candidate key (*not a superkey*) for R. Sometimes stated as "A is *prime*"

If R is in BCNF, then it is in 3NF. R can always be decomposed (lossless and preserving dependencies) into a collection of 3NF relations

If R is in 3NF, some redundancy is possible, but at least dependencies are preserved.

### Decomposing into 3NF

* Follow the BCNF algorithm
* To ensure dependency preservation, if X -> Y is not preserved, add relation XY (but this might violate 3NF!). So do this carefully.
* Refinement - to avoid this, don't use the given set of FDs F. Use a *minimal cover* of F to do the decomposition algorithm.
  * Minimal cover - a minimal cover G of F is an FD such that G+ = F+, but G is the smallest set of FD's possible.

If you decompose into 3NF, or into BCNF (non-dependency-preserving), you will have to handle the redundancy or dependency preservation in either CHECK constraints or Application Logic. This is tough.

Functional dependencies appear in a lot of data applications, in a variety of forms. Look out for them!



















