# Chapter 2 - Data Models and Query Languages

*07 Dec 2019*

These are my notes from the second chapter of Martin Kleppmann's: Designing Data Intensive Applications.

I've edited it slightly to make the headings more consistent and generally make it easier to read.

* * *

This chapter surveys a couple of data models for storing and querying data, as well as different query languages.

## **Relational Model Versus Document Model**

SQL is the best known data model today. Two of the earlier competitors of this model were the *network model* and the *hierarchical* model. NoSQL is the latest attempt to overthrow SQL's dominance. Some of the driving forces behind NoSQL's dominance are:

- A need for greater scalability than relational databases can easily achieve, including very large datasets or very high write throughput
- Preference for free & open source software over commercial DB products.
- Some specialized query operations not well supported by the relational model.
- Frustration with the restrictiveness of relational schemas, and a desire for a more dynamic and expressive data model.

NoSQL Databases are in two forms:

- **Document databases**: Targets use cases where data comes in self-contained documents and relationships between one document and another are rare
- **Graph databases:** These go in the opposite direction, they target use cases where anything is potentially related to everything.

## **Impedance Mismatch**

An awkward translation layer is required between objects stored in relational tables and the application code. The disconnect between the models is called an *impedance mismatch.* ORMs usually help with this but they don't hide all the differences between both models.

Some developers feel that the JSON model reduces impedance mismatch between the application layer and the database layer.

While relational models refer to a related item by a unique identifier called the *foreign key,* document models use the *document reference.*

## **Relational Versus Document Databases Today**

Both data models have arguments going for them. For the relational model, it provides better support for joins, and many to one and many to many relationships.

The document data model has the advantages of schema flexibility, better performance due to locality, and for some applications, it's closer to the data structures used by the application.

If data in the application has a document-like structure (i.e. a tree of one-to-many relationships, where typically the entire tree is loaded at once), document model is a good idea. Document databases typically have poor support for many-to-many relationships. In an analytics application, for example, many to many relationships may never be needed.

*Basically, for highly interconnected data, the document model is awkward, the relational model is acceptable, and graph models are the most natural.*

## **Schema Flexibility in the document model**

Document databases have the advantage of having an implicit schema which is not enforced by the database: Also known as *schema-on-read -* The structure of the data is implicit, and only interpreted when the data is read. This is contrasted with *schema-on-write:* The schema is explicit and database ensures all written data conforms to it.

## **Data locality for queries**

There is a performance advantage to data locality in the document model if you need to access large parts of the document at the same time. Compared with many relational databases where data is usually spread among tables, there is less need for disk seeks and it takes less time.

There are a couple of tools nowadays that offer this locality in a relational model e.g. Google Spanner, Oracle (multi-table index cluster tables), Cassandra and HBase (column-family concept).

## **Convergence of document and relational databases**

Relational databases like PostgresSQL, MySQL and IBM DB2 have added support for JSON documents in recent times.

Document databases like RethinkDB also support relational-like joins in its query language.

These models are becoming more similar over time.

## **Query Languages For Data**

There is a distinction here between imperative query languages and declarative.

An imperative language tells the computer to perform certain operations in a certain order. A declarative language specifies the pattern of the data wanted, and how the data should be transformed, but not *how* to achieve that goal.

An advantage of this declarative approach to query languages is that it hides the implementation details of the database engine, which makes it possible for the database system to introduce performance improvements without requiring any changes to query.

*HTML and CSS are also declarative languages.*

## **MapReduce Querying**

MapReduce is a paradigm for querying large amounts of data in bulk across many machines. Basically, map jobs are run on all the machines in parallel, and then the results are reduced.

It's neither a declarative query language nor a fully imperative query API, but somewhere in between. The logic of the query is expressed with code but it's repeatedly called by the processing framework.

Both the *map* and the *reduce* functions must be pure functions, they only use data passed to them as input, and they cannot perform additional database queries, and they must not have any side effects.

*MongoDB has implemented aggregation pipelines which is similar to MapReduce.*

## **Graph-Like Data Models**

If many to many relationships are common, the Graph model is probably best suited for it.

Graphs are not limited to homogenous data. Facebook, for example, maintains a single graph with many different types of vertices and edges. Their vertices represent: people, location, events, checkins, comments made by users. Their edges indicate which people are friends with each other, which checkin happened in which location, who commented on what, who attended which event etc.

There are a couple of different but related ways for representing graphs:

- Property graph model: Neo4j, Titan and InfiniteGraph
- Triple-store model: Datomic, AllegroGraph

Some declarative languages for querying graphs are: Cypher, SPARQL and Datalog.
