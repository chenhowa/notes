# SQL: Query Language

Codd's Theorem - relational algebra and relational calculus have the same expressive power. That is, a declarative calculus query can be transformed into an operational algebra query.

SQL deals with multisets rather than sets during the transform to operators. SQL is more powerful than the calculus and the algebra.

## Starting from an SQL Query

From a single SQL Query, the optimizer will produce multiple algebras (composed operators), and then choose the cheapest one.

There are two sublanguages within SQL. The DBMS is responsible for executing these sublanguages efficiently.

* DDL - Data Definition Language
  * Define and modify schema
* DML - Data Manipulation Language
  * Query the data

### DDL

* CREATE TABLE
  * PRIMARY KEY - unique identifier field(s). Can be a composite primary key.
  * FOREIGN KEY - specifies that a column references the primary key of another table.

### DML

* SELECT
  * DISTINCT - specify that duplicates should be eliminated
  * FROM - allows specification of table (can be more than one, if some Join behavior is desired)
  * WHERE - specifies the condition

To imagine the result of a SQL query, look at the clauses in the following order

1. FROM
2. WHERE
3. SELECT 
4. GROUP BY
5. HAVING
6. DISTINCT

### Arg-Max Query

What if we want the Sailor with the highest rating? It can be done without aggregate functions, but it's much easier with aggregates.

But the best way is to sort all the sailors by descending rating, and limit the output to just 1. Note: You cannot use LIMIT in subqueries.

### Null Values

The "undefined" value. This has syntax "IS NULL" and "IS NOT NULL". It also requires new syntax (Outer Join). Booleans now have 3 values: True, False, and Unknown, since comparisons to Null will often have the value Unknown

In SQL, Nulls are eliminated before the aggregation is done.

## Joins

* INNER JOIN - the same as the Theta Join, and the Natural Join
  * Natural join means do an Equi-Join on each pair of attributes with the same name. This is hard to maintain, since if someone accidentally adds a column with the same name, the queries start looking weird.
  * Inner Joins tend to drop rows silently. Outer joins thus tend to be more useful.
* LEFT OUTER JOIN - return all matched rows, plus all unmatched rows from the left table, with null values for all columns from the right table
* RIGHT OUTER JOIN
* FULL OUTER JOIN - keep all unmatched rows from the left table, and from the right table, with null values for all the missing columns

## Views and Subqueries

* CREATE VIEW
  * Name your query
  * Can use the VIEW in other queries
  * Not materialized at first -- not stored on disk; lazily evaluated.
  * Often used for security -- you can have access to a VIEW of a table, but not to the full table itself.
  * You can also just inline your view in the FROM clause, if you don't want to create a VIEW.
  * You can also use the WITH clause to add a subquery, right above the SELECT.

## Discretionary Access Control

* GRANT privileges ON object TO users
  * Privileges - select, insert, delete, references, all
  * Privileges can be REVOKE'd.
  * Users can be single users or groups.
  * Objects can be Tables or Views.

## Constraints

Integrity Constraints are Boolean expressions that every row of a relation must satisfy.

* Domain constraints - field values must be of the right type
* Primary key constraints - primary key columns must be a unique identifier
* Foreign key constraints - foreign key columns must reference real records in the foreign table
* General constraints - that a column's value can never be less than 0, or be NULL, etc.

### Primary Keys

* Superkey - no two distinct tuples can have the same values in all key fields.
* Key - the set of fields is a Superkey, and no subset of the set is also a superkey.
* There may be multiple possible keys for a relation, but only one can be chosen as a primary key in the DBMS.

## Embedding SQL in PLs

See next lecture


















