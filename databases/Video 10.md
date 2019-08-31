# More on SQL

## Constraints

Constraints are described in the SQL DDL. Since they can be put in the database itself, the business logic of the applications does not need to worry about maintaining the constraints.

### Integrity Constraints

Every legal instance of a relation must satisfy the ICs on a relation. Affects inserts/updates/deletions.

* Domain constraints - field values must have the right type
  * NOT NULL
  * Type of the field
* Primary/Foreign key constraints - primary keys must be unique and non-empty. Foreign keys must reference *existing* primary keys from the other tables (no dangling pointers). Deletion of records from the foreign table will require a lookup for dangling pointers in the foreign key field -- do we make them null, or do we cascade delete?
  * CASCADE delete - on delete of the foreign record, delete the records with the dangling pointer
  * NO ACTION - actions that use this foreign key are not allowed, due to dangling pointer
  * SET DEFAULT - on delete, set to a default value
  * SET NULL - on delete, set to null
* General constraints - constraints on the values of the fields of the tuple.
  * uses the CHECK keyword in the DDL.
  * CHECKed on insert or update
  * In some systems, queries can be used to express the general constraints
  * General constraints can be named - some systems let you to alter, disable, enable, or drop constraints based on the names.
  * To have constraints on a table that utilizes multiple other relations, you cannot use CHECK
    * Use ASSERTION (if supported), or Triggers

### Keys

Because keys can uniquely identify records in a relation, they can be used to associate tuples in different relations. If there is more than 1 key for a relation, then one is chosen as the *primary key*, and the rest of the keys are called the relation's *candidate keys*, which are specified using the "UNIQUE" keyword in the DDL.

## SQL Embedded in Other Languages

SQL is not a general-purpose programming language; it's a DSL. It's easy to optimize and parallelize, but it sacrifices power for that. So what do we do if we want to do more?

* Make it Turing complete.
* Let SQL be embedded in regular programming languages.
  * Cursors
  * Database APIs

### Cursors

In the host language (Java, Python, etc.) Cursors are declared in relations or queries. You *open* the cursor, and then you iterate over query results using the cursor. You can also modify/delete tuples pointed to by a cursor, *if* the cursor is actually pointing at a tuple.

### Database APIs

These are special objects/methods in the host language that present the result sets in a language-friendly way, while hiding the actual DBMS being used.

* ODBC - C/C++ standard for this
* JDBC - for Java
* Object-Relational Mappings - Ruby on Rails, Django, Spring -- automatically turn database rows into programming language objects, that can be manipulated directly.
  * Loss of power for increase in convenience
    * Performance tends to be worse for complex queries
    * Developer has less control compared to using SQL directly.
* Future directions - embed SQL in a host language, but make it extremely easy to use both SQL and the host language.


















