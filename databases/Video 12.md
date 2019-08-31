# ER Model Basics

* Entity - real world "object" with a set of attributes
* Entity Sets - collection of entities with the same attributes.
  * Each entity has a "key" that uniquely identifies each entity.

* Relationship - association among two or more entities
  * Can have their own attributes. Each attribute has a domain (type) of valid values.
* Relationship Set - collection of similar relationships. Relates one entity set with another entity set.
  * Arity - number of entity sets that are associated by the relationship set.

The same entity set can participate in multiple relationship sets, or in different the same relationship set *more than once*. For example, two employees participating in the "manages" relationship.

Key constraints in relationship sets

* Many to Many
* Many to (at most) One; (at most) One to Many
* One to One

Participation Constraints

* Total - every entity participates in the relation
* Partial - every entity may participate in the relation.

## Weak Entity

A weak entity can be identified uniquely only be considering the primary key of another (owner) entity.

Weak entity must have *total participation* in the "identifying" relationship set.

For example, the parent-child relationship. Parents are the owner entity, and the child may have only a "name" and "age" attribute. The only way the child is uniquely identified is through the "child of" relationship -- through the parent.

The full key of a weak entity set is the (parent key, partial key of the weak entity). For example, for a child, the partial key is probably the name of the child.

## Binary and Ternary Relationships

The meaning of ternary relationships is less clear, so it has to be thought out carefully. Both the participation and key constraints need to be carefully thought out. In many cases, two binary relationships are more appropriate than one ternary relationship. But the reverse can also be true.

## Aggregation 

In ER Models, this involves wrapping up entities and relationships into one large entity sets. The rough idea is to allow relationships between relationships. Which makes sense; perhaps an entity needs to be associated with multiple entity sets that are in a particular relationship.

It's a kind of encapsulation.

You can try to emulate aggregation with an n-ary relationship. However, the semantics are typically not correct when you do this, which can lead to compliance/audit problems with the data.

## Conceptual Design with the ER Model

Design choices

* Entity or attribute?
  * Should an address be an attribute of employees, or an entity of its own?
    * How many addresses per employee?
    * Do we care about storing the structure of an address (city, street, etc.)?
* Entity or relationship?
* Binary, ternary, or aggregation?

The goal is to capture as many semantics of the real world as possible (compliance is a big deal with the database)

But some constraints cannot be captured in ER, like constraints on the domains of attributes.

NOTE: In the ER model, a relationship set between entity sets implies that the relationship can only occur *once* between two particular entities. You cannot have duplicate relationships between entities.

NOTE: To express a 1 - 1 relationship, the best way is probably a many-to-many table where each foreign key is unique. This avoids the need for any validation rules.

## Converting ER Schemas to Conceptual Schemas

Entity Sets

* Key of the entity set is the key of the corresponding relation (table)

Relationship Sets

* Key constraints
  * If many to many, just create a table that has foreign keys to the entities that it's connecting, as well as any other attributes.
  * If one to many, you can create a table with the foreign keys of the entities, but just make the "one" entity the primary key, so that it can occur only once in the relation.
    * Or, you can combine the relationship table with the "one" relation, and give it a field that references the "many" relation. *This is less flexible if the key constraints change (like if it becomes many to many)*, but it does require less lookups.
* Participation Constraints
  * Use "not null" to enforce participation, in a one-to-many relationship, *if* you combined the relationship table with the "one" relation.
  * But if you can't do this (like in a ternary relationship, or a total one-to-one relation), this doesn't work. You would need a "CHECK" constraint to reject the entity if they are not participating by the end of the transaction. The problem with "CHECK" constraints is that they are very expensive to enforce...but sometimes they're necessary.
* Weak entities
  * Combine the weak entity and the relationship into one table, and make the primary key the combo of the owner's primary key and one of the weak entity's fields.
* Aggregations
  * FIGURE OUT HOW TO TRANSLATE AGGREGATIONS
* Ternary Relationship Sets
  * FIGURE OUT HOW TO TRANSLATE THESE
* ISA hierarchies - the inheritance model in ER Models
  * LOOK UP, and then translate.

ER design can be subjective. It requires analysis and refined as much as possible to get the most correct semantics for the data. Further topics: Functional Dependencies and Normalization.












