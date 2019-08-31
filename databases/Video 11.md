# Window Functions and Data Modeling

Many Spark features are available for PostgreSQL too, as well as for other commercial DBMS's.

## Window Functions

Order-based operations don't work well in SQL, because SQL is set-based.

SQL's solution to this is Window Functions, which impose order on SQL queries. This is useful for doing statistical analysis with SQL.

## Data Modeling: ERDs and Relational Models

* Data model: Collection of concepts that provide a way of describing the data
* Schema: description of a collection of data, using a given data model
* Relational model: a type of data model, that describes data using a relation (table), rows, and columns. Each relation has a schema that describes the columns, the column names, and the domains of the columns.

### Levels of Abstraction

* Views - what a given user can see from a relation's data.
  * Conceptual Schema - columns, domains, constraints
    * Physical Schema - what disk is it on, what indexes are available, what file types are used, etc.
      * Disk

## Database Independence

Before the relational model, applications over databases tended to follow disk pointers. Reorganizing the disk broke apps.

With data independence, apps are insulated from the data structure

* Logical - protection from changes in the conceptual schema. Hidden using views.
* Physical - protection from how data is stored on disks. Hidden using the conceptual schema.

These are important to have because Database Applications stick around forever. So databases evolve faster than the applications, so the applications need to be insulated from the Databases.
(dApp/dT) &lt;&lt; (dEnv/dT)

## Entity Relationship Data Model (ERD)

The ERD is the most widely used model, although many other models exist:

* XML, JSON (nested, semi-structured -- hard to work with)
  * XQuery, JSONic
* Object Relational Mergers (foundation of ORMs)

The ERD is a graphical abstraction over the relational model.

### Steps in Traditional Database Design

* Requirements Analysis
* Conceptual Design (often done with ERD)
  * What are entities, relationships?
  * What integrity constraints ("business rules?")
* Logical Design (translate ERD into DBMS data model)
* Schema Refinement (improvements)
* Physical Design (indexes, disk layout of schema)
* Security Design (who accesses what, and how)
  * What views are exposed, and to whom? Which views allow for updates, for deletes, etc.

### Modern Design: "Schema on Use"

If the environment is less governed, and can be more agile, then don't let the lack of a full design prevent the storing of data. Use binary, text, CSV, JSON, etc, and shove that data into a database or filesystem, because most database engines can query files directly these days.

* Define views over raw, unstructured data, even though no conceptual schema yet exists.
* Integrity constraints don't exist -- just define "anomaly indicators". You allow errors, because you're agile, but you allow reporting on the weird data.
* This fits well with read/append-only data. It does NOT fit well with update-heavy data, because the updates may not make sense (see future lecture)
* "Loosely typed"

### ER Model Basics

Entity - real world object described by a set of attributes

Entity Set - collection of similar entities. They all have the same attributes. Each entity set has a *key* to identify individual entities. Each attribute has a *domain*

Relationship - association among two or more entities. Each association can have its own attributes.

Relationship Set - collection of similar relationships. They all have the same attributes and associate the same entity sets.


























