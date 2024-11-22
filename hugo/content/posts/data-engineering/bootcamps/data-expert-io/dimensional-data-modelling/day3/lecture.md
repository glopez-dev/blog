---
date: 2024-11-19T17:46:24+01:00
draft: false
author: Gabriel LOPEZ
title: (DataExpert.io) Bootcamp - Day 3 - Lecture 
---
Today's lecture is about dimensional additivity and how to build a flexible data model ready for graph database consumption

## Index
- Additive VS non-additive dimensions
- The power of Enums
- When should you use flexible data types ?
- Graph data modeling

## Additive vs Non-additive dimensions
### What makes a dimension additive ?
Additivity refers to whether numerical facts (measures) in a fact table can be meaningfully aggregated across different dimensions.

If you take all the sub-totals and sum them up you should have the total

*Example :* The sum of the count of people of each age gives the total population  

An additive dimension is one that don't "double count"

*Example :* The total number of driver doesn't equals the sum of each car brand drivers because some people owns two cars

> You should ask yourself if a user can be two of these at the same time on a given day.

### The essential nature of additivity
A dimension is additive over a specific time window, if and only if, the grain of that window can only be one value at a time.
- In a time window of one second, one driver can't drive two cars
- In a time window of one day, one driver can drive many cars

### How does additivity helps ?
You don't need to use `COUNT(DISTINCT)` on preaggregated dimensions.

> **Note :** Non-additive dimensions are usually only non-additive with respect to `COUNT` aggregations but not `SUM` aggregations.


## The power of Enums
### When should you use enums ?
Enums are great for low-to-medium cardinality

*Example :* countries (200) might be to much for an enum.

If it's less than 50 it might be a good enum

### Why should you use enums ?

#### Built in data quality
- The Enum guarantees the set of values a field can take.
- This prevents invalid values at the database level
- You can't insert 'pending ' (with space) or 'PENDING' (wrong case)
- More reliable than `CHECK` constraints on `VARCHAR` columns

#### Built in static fields
- The enum values are stored as internal constants in the database
- More storage efficient than VARCHAR
- PostgreSQL stores enum values as integers internally but presents them as strings to users
```sql
-- More efficient than storing the full string each time
SELECT COUNT(*) FROM orders GROUP BY order_status;
```

#### Built in documentation
- The enum definition serves as documentation in the database schema
- Developers can see all valid values by checking the enum type:

```shell
# Postgres SQL command :
\dT+ status
```

### Enumerations and subpartitions
Enumerations makes amazing subpartitions because :
- You have an exausthive list
- They chunk up the big data problem into manageable pieces

*Thrift* is a tool to manage schemas in your logging as well as you ETLs

>  https://en.wikipedia.org/wiki/Apache_Thrift

Zach created a framework named "Little book of pipelines"

>  https://github.com/EcZachly/little-book-of-pipelines

The enum pattern is useful whenever you have tons of sources mapping to a shared schema.

## How do you model data from disparate sources into a shared schema ?

You don't want to bring all the columns from 40 tables into one table with 500 columns 
- Most columns are going to be NULL all the time
- It's not really usable table

You want to use a "flexible schema"
- Use a lot of map types
- Overlaps a lot with graph database
	- The enumerated list of things is similar to a "Vertex type"

### Flexible schemas

If you need to add more things you just add it to the map

If new columns appears you can add them to the map

> **Limit :** Maps in Spark can have up to 65,000 keys

#### Benefits
- No `ALTER TABLE` commands
- You can manage a lot more columns
- You don't have a ton of `NULL` columns
- You can use an "other_properties" 
	- Carry rarely used but needed column 
	- Allows you to ignore them during modelisation

#### Drawbacks
- Compression is terrible 
	- Map is one of the worst compression of all data types
	- The only thing worse is JSON
	- The column headers becomes map keys so they are part of the data instead of the schema and duplicated in each row

## How is graph modelling different ?

Graph modelling is Relationship focused, not Entity focused.

You can do a very poor job at modeling entities

### Zach super secret sauce for Graph data modelling :
He always met the same schema when doing graph databases

We don't care about columns

Usually for an **entity** in a graph the model is :
```sql
Identifier: STRING
Type: STRING
Properties: MAP<STRING, STRING>
```

The **relationships** are modeled a little bit more in depth;

Usually the model looks like :
```sql
subject_identifier: STRING
-- Example: player_name
subject_type: VERTEX_TYPE 
-- Example: player
object_identifier: STRING
-- Example: team_name
object_type: VERTEX_TYPE 
-- Example: team
edge_type: EDGE_TYPE
-- Example: "PLAYS WITH" , "PLAYS AGAINST" (an action)
properties: MAP<STRING, STRING>
```