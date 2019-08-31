# Functional Dependencies and Schema Refinement

Consistency and normalization considerations.

## The Evils of Redundancy

Redundancy in relational schemas leads to wasted storage, and anomalies with insert/delete/update, because there are consistency problems among repeated data.

The solution is *Functional Dependencies*

* A form of integrity constraints
* Helps identify redundancies in schemas
* Helps suggest refinements

The main refinement technique is *Decomposition*

* Split the columns of one table into two tables (that are less wide).
* This is often good, but needs to be done with thought to the cost-benefit analysis. Too many tables with too-few columns is hard to manage; hard to query.

## Functional Dependencies

X -> Y means that

* Given *any* two tuples in table R, if they have matching values in columns "X", then they have matching values in columns "Y".
* For example, suppose we have two columns; Country and Nationality. Well, then perhaps Country -> Nationality.
* Functional Dependencies are not learned from the data; they are specifies as constraints. It's a design decision.

Thus, we can say that the key of a table determines the values of all columns. That is, all columns have a functional dependency on the key of a table.

We can also  say that column x has a functional dependency on itself (trivial dependency)

### Problems with Functional Dependencies

* Update anomalies - updating the dependent column may require updating tons of other records with the same independent column.
* Delete anomalies - when you delete records, you may lose  the functional dependency entirely (i.e., no records show it in the database).
* Insert anomalies - when you insert records, how do you know whether it satisfies the functional dependency? You have to do it either in code, or in the structure of the database. It is better to not do it in code.
* Functional dependencies are not a problem if the independent column is unique. Then the anomalies don't matter, since the dependency is per record, so their is no redundancy.

### Decomposing a Relation

Functional Dependencies will show us how to chop the relations into pieces.

* if R -> W, then pull R and W into a separate relation, and add a column that has a foreign key to the separate relation.

## Chaining FDs

* title -> studio, star implies title -> studio and title -> star
* title -> studio and studio -> star implies that title -> star 
* But title, star -> studio does NOT imply that title -> studio or star -> studio

We say that an FD g is *implied by* a set of FD's F if g holds whenever all the FDs in F hold.

F+ = the closure of F - the set of all FDs that are implied by F (including trivial dependencies)

### Rules of Inference

Armstrong's Axioms. Assume that X, Y, Z are sets of attributes. You can use these axioms to generate F+ (the closure of F)

* Reflexivity: If X is a subset of Y, then X -> Y
* Augmentation: If X -> Y, then X U Z -> Y U Z for any Z.
  * if Rating -> Wage, then Rating, LastName -> Wage, LastName
* Transitivity: If X -> Y and Y -> Z, then X -> Z.

Additional rules can be derived:

* Union: If X -> Y and X -> Z, then X -> YZ
  * if Rating -> Wage and Rating -> Bonus, then Rating -> Wage U Bonus
* Decomposition: If X -> YZ, then X -> Y and X -> Z
  * if Rating -> Wage U Bonus, then Rating -> Wage and Rating -> Bonus

### Attribute Closure

Computing the closure F+ of a set of FD's F is hard -- exponential in the number of attributes.

Typically, just check if X -> Y is in F+. So all we need to know is the attribute closure of X (X+) with respect to F.

* X+ = Set of all attributes A such that X -> A is in F+.
  * X+ := X
  * Repeat until no change 
    * if U -> V is in F, and U is a subset of X+, then add V to X+
  * Then check if Y is in X+ by the end of it.
* To find the keys of a relation,
  * if X+ = R, then X is a superkey for R.
  * Try dropping each element of X+, and see if X+ is still a key.

## Using Functional Dependencies for Schema Refinement

### Normal Forms

Is any refinement needed? If a relation is a normal form, then we know certain problems are avoided/minimized. The benefits of each normal form helps us decide whether decomposing a relation has provided enough benefit.

BCNF is the goal; 3NF is second best.

Consider a relation R with attributes A, B, C:

* Suppose there are no FDs, so no redundancies.
* Suppose A -> B.
  * If A is a key, then still no redundancy.
  * If A is not a key, then there is a redundancy -- tuples with the same A value will have the same B value.

Normal Forms

1 is a subset of 2 is a subset of 3 is a subset of Boyce-Codd, and so on.

* 1st Normal Form -- all attributes are atomic
  * This is violated by many other data models, which makes it hard to reason about functional dependencies.
  * Non-first-normal-form (NFNF) is useful in various settings, like never-update, fixed-query settings, where there is never a need to "unnest" to join, then. Transport of data is one example of this, where JSON and XML make sense.
* 2nd Normal Form (skipping)
* 3rd Normal Form
* Boyce-Codd Normal Form
  * Take a relation R with FD's F. If, for all X -> A in F+,
    * A is a subset of X (a trivial FD), or 
    * X is a superkey for R
  * In other words, if the only non-trivial FDs over R are key constraints.
  * Each column is essential.
  * No inferring happens except trivial inferences. That is, given an FD X -> A for a relation, if you can guess the value of the A column given a value from the X, then you aren't in BCNF.
















