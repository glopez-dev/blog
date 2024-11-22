---
date: 2024-11-15T17:46:24+01:00
draft: false
author: Gabriel LOPEZ
title: (DataExpert.io) Bootcamp - Day 1 - Lecture 
---

> Today lecture with deal with **complex data types** and **cumulation**

### What is a dimension ?

Dimensions are the attributes of an entity

- Some dimensions are identifiers
- Some dimensions are just attributes

Dimensions come in two flavors (generally) :
- Slowly changing (time dependent)
	- Makes things harder to model
- Fixed (doesn't change over time)

## Topics of the day (index)
- Knowing your data consumer
- OLTP vs OLAP modelling
- Cumulative table design
- The compactness vs usability tradeoff
- Temporal cardinality explosion
- Run-length encoding compression gotchas

## Knowing your consumer
Who is going to consume the data ?

**Data analyst / Data scientist :**
- Data should be easy to query
- Not many complex data types
- OLAP cube ?

**Other data engineers :**
- Data should be compact
- Probably harder to query
- Nested types are okay
- Master dataset >> consumed by others D.E

**M.L models :**
- Most models use identifiers and primitive types columns
- "Flat data"

**Customers :**
- Easy data interpretation
- Data visualization

## OLTP vs Master Data vs OLAP

**OLTP** >> On Line Transaction Processing
- Optimizes for low-latency, low volume queries
- Mostly outside of Data Engineering realm
- It is how Software Engineers model their data to make their online systems run quickly
- 3NF, deduplication

**OLAP** >> On Line Analytical Processing
- Optimizes for large volume, `GROUP BY` queries, minimize `JOIN`s
- Most common data modelling for D.E
- Big JOINs are slow

OLAP usually looks at a big chunk of the dataset while OLTP looks at one record
 
**Master Data** 
- Middle ground between OLTP and OLAP
- Transactional layer > Master Data layer > Analytical layer

**Mismatching the needs leads to less business value**
- Biggest D.E problems occurs when data is modelled for the wrong user
Symptoms :
- Analytical modelling used as Transaction >> Online app will be slow
- Transactional modelling for analytics >> Lot of JOINs
- That's were master data middle role helps a lot

**OLTP and OLAP is a continuum**

![oltp_olap_continuum](/images/oltp_olap_continuum.png)

- OLAP Cube 
	- often reffered as "slice and dice"
	- flattened data
- Metrics 
	- aggregates even more 
	- OLAP cube to one value

*The continuum is like "distillation" you go from lots of productions database to one metric*

Understanding these patterns in data modelling will simplify the D.E life

## Cumulative Table Design
> https://github.com/DataExpert-io/cumulative-table-design

This design produces tables that can provide efficient analyses on arbitrarly large (up to thousands of days) timeframes.

We initially build our **daily** metrics table that is at the grain of whatever our entity is. This data is derived from whatever event sources we have upstream.

After we have our daily metrics, we `FULL OUTER JOIN` yesterday's *cumulative table* with today's daily data and build our metric arrays for each user. This allows us to bring the new history in without having to scan all of it. **(a big performance boost)**

These metric arrays allow us to easily answer queries about the history of all users using things like `ARRAY_SUM` to calculate whatever metric we want on whatever time frame the array allows.

> The longer the time frame of your analysis, the more critical this pattern becomes!!

**It answers common problems when you build master data :**
- All users may not always showup
- You still want a complete history
- Holding on all the dimensions that ever existed (history)

**Example :**
- Two Dataframes (yesterday and today)
- `FULL OUTER JOIN` the Dataframes together
- `COALESCE` values to keep everything around
- Hang onto all of history

### Use cases
-  Growth analytics
- State transition tracking
	- Growth accounting : 
		- Active yesterday but not today (churned)
		- Inactive yesterday but active today (resurected)
		- Not existing yesterday but exists today (new)	

### Stengths
- Ability do to historical analysis
	- No need of GROUP BY
- Scalable queries

### Drawbacks

#### Sequential Backfilling 
*Backfilling* means populating historical data retroactively. In this design, you can't load historical data in any random order - you must process it sequentially (day by day)
#### Handling *Personal Identifiable Information* (PII)
 If **PII** needs to be updated or deleted (e.g., for privacy regulations like GDPR):
- You can't just update the latest record
- You need to update/remove PII from the entire history in the arrays
- This could break the historical analysis capabilities

## Compactness vs Usability tradeoff

**The most usable tables usually :**
- Have no complex data types
- Easily can be manipulated with `WHERE` and `GROUP BY`
- More analytics focused

**The most compact tables :**
- ID + blob of bytes
- Use compression codex
- Not human readable
- Ex : for transmission over Network 
- More Software engineering focused

**Middleground tables :**
- Use complex data types (ARRAY, MAP, STRUCTS)
- Querying is thickier
- Data is more compact

**Where to use each type of table ?**
Most compact :
- Online systems
- When latency and volumes matters a lot
- Highly technical consumers
 
 Middle-ground :
- Upstream staging / master data
- Mostly D.E consumers
 
 Most usable :
- OLAP cubes
- Analytics consumers

### Struct vs Array vs Map
#### Struct
- Almost like a table inside a table
- Keys are rigidly defined
- Compression is good
- Values can be any type
#### Map
- Keys are loosely defined
- Compression is okay
- Values have to be the same type
#### Array
- Ordered values
- Values are all the same type
- Values can be another complex type (Struct, Map)

## Temporal Cardinality Explosions of Dimensions

> One of the most impactful things Zach worked on.

When you add a temporal aspect to your dimensions and the *cardinality* increases by at least one order of magnitude.

**Example :** 
- Airbnb has around 6 millions listings
- We want to know the nightly pricing and availability of each night for the next year
	- 365 days * 6 million listings ~ 2 billions nights
- Should this dataset be :
	- Listing-level with an array of 365 nights (6 million rows) ?
	- Listing night level with 2 billion rows ?
- If you do the sorting right Parquet will keep these two datasets about the same size

### Badness of denormalized temporal dimensions
- If we choose the Listing night level Spark `JOIN` shuffling is gonna mess with the sorting
- Other run-length encoding is not going to compress that down as much
### Run-Length Encoding (RLE) compression
Probably **the most important compression technique in Big Data** right now.

> It's why Parquet file has become so successful

Shuffle can ruin *RLE* >> **Be careful!**

 > Shuffle happens in distributed environments when you use `JOIN ` and `GROUP BY`.

Big thing about *RLE* is that duplicated data gets nullified

After a join, Spark may mix up the ordering of the rows and ruin RLE compression.

| season | player_name     | age | height | college         |
|--------|----------------|-----|--------|----------------|
| 1996   | A.C. Green     | 33  | 6-9    | Oregon State   |
| 1996   | Michael Jordan | 34  | 6-6    | North Carolina |
| 1997   | A.C. Green     | 34  | 6-9    | Oregon State   |
| 1997   | Michael Jordan | 35  | 6-6    | North Carolina |


#### Two ways to solve the problem :
- One way to go is to re-sort the dataset after a join 
	- Zach is not about that; you should sort your data once
- Store everything in an array
	- A row with player name and seasons in an array
	- `JOIN` on `player name`
	- Explode the seasons array after the join 
	- It keeps the sorting because rows with the same player name are kept together after the explode

Leveraging this for master data can be powerful, especially for downstream data engineers
- You model data the right way so they can't make that mistake

**Spark shuffles are a big thing to watch out for**


