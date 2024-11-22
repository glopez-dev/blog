---
date: '2024-11-17T19:21:32+01:00'
draft: false 
title: (DataExpert.io) Bootcamp - Day 2 - Lecture
---
Today's lecture deals with **Slowly Changing Dimensions** and **Idempotency**.

> **Slowly changing dimensions** = An attribute that drifts over time

*Example:* Your favorite food 

## Idempotency

You need to model slowly dimensions the right way because they impact idempotency.

> **Idempotent** = Denoting an element of a set which is unchanged in value when multiplied or otherwise operated on by itself.

> **Idempotent pipeline** = The ability for your data pipeline to produce the same results whether it's running in production or in backfill.

It's a very important property of pipelines in order to enforce data quality

If you can build **idempotent** pipelines you are going to be a much better Data Engineer.

**Your pipelines should produce the same result :**
- Regardless the day you run it 
- Regardless of many times you run it 
- Regardless of the hour that you run it 

> Data Engineers mistakes a lot these three characteristics of a pipeline. 

**Why is it hard to troubleshoot a non-idempotent pipeline ?** >> It doesn't actually fails, it just produces incorrect data silently.

> Zach prefers to say "non reproducible" data whether "incorrect"

If your data isn't idempotent downstream data will also be (there's propagation of the problem) 

## What can make a pipeline not idempotent ?

**Using `INSERT INTO` without TRUNCATE**
- Creates duplication
- Your pipeline doesn't produces the same data regardless how many times it's run
- `INSERT INTO` should never be used as a Data Engineer
-  `MERGE` and `INSERT OVERWRITE` are better ways to go
	- `MERGE` avoids duplication


**Using `start_date >` without a corresponding `end_date <`**
- While the time passes further new runs of the pipeline are going to add data from longer time periods.
- You need to use start and end dates to process only a "window"
- This can also cause out of memory exceptions when you backfill

**Not using a full set of partitions sensors**
- Your pipeline might run with an incomplete set of inputs
- You have to check for the full sets of input needed by the pipeline
- Avoid to fire the pipeline to early in production

**Not using `depends_on_past` (Airflow term) for cumulatives pipelines**
- Most pipeline can run in parallel
- Cumulatives pipelines have to run sequentially

> The production and back-fill behavior of a pipeline have to be the same.

> **SCD** = Slowly Changing Dimensions

**Relying on the "latest" partition of a not properly modeled SCD table**
- Cumulatives pipelines amplifies this bug
- The only exception for using "latest" partition is back-filling data 
- The SCD table has to be properly modeled though

**Relying on the "latest" partition in general**

> You can't do back-filling without idempotency

> Unit tests doesn't guarantees the pipeline idempotency.

## Should you model as Slowly Changing Dimensions ?

There is also a concept of **rapidly changing dimensions**.

> Example: The heart rate

**The creator of Airflow hates SCDs :**
- He defends the idea of *functional data engineering*
	- The pipeline is a chain of pure functions
- As the storage is cheaper he prefers storing each version of the data no matter duplication.
	- Daily dimensions values over SCDs

### Three different ways to model your dimensions

#### Latest snapshot
You just store the current value.

**Weakness :** If you have SCD and you only hold-on to the latest value the pipeline is not idempotent.

You are going to back-fill  values with the latest data version sometimes with a huge time gap.

That's where daily snapshot is better for back-filling

#### Daily / Monthly / Yearly snapshot

#### Slowly Changing Dimensions
A way to collapse daily snapshot based on whether or not the data changed day over day.

>Example : One row saying you have this age for this time frame (1 years) instead of 365 rows with the same age.

The slower the dimension changes the better is the compression.	

> Saving 7 rows (a week) with compression is less interesting than saving 365 rows (a year).

Keep this in mind when choosing your modelling.

## How can I model dimensions that change ?

There are three ways to model SCD :

1. Singular snapshot
2. Daily partitioned snapshots (Easy and intuitive way to go)
3. SCD types 1, 2, 3 (three subways)

> **Warning :** Never back-fill with only the latest snapshot

## The types of SCD
### Type 0
Not changing dimension 

### Type 1
- You only care about the latest value
- This breaks your pipeline idempotency
- For OLTP it can be fine
 
### Type 2
You care about what the value was in a certain time frame.

> `start_date` and `end_date`

Current values usually have either an `end_date` that is `NULL` or far in the future

**Benefits :**
- Only type of SCD that is purely idempotent.
- Full change history

**Drawback :**
- There are multiple rows for the same dimension
	- You'll have to filter by date


### Type 3 
You only care about "original" and "current" values

**Benefits :**
- You only have 1 row per dimension
- No row filtering

**Drawbacks :**
- You loose the history in between "original" and "current".
- It may not store when the value changed unless you have a `current_change_date`
- It is not idempotent

> There are others types of SCD models that are less used.


## Loading Type 2 SCDs

Two ways to load a table with SCDs :
1. Load the entire history in one query
	- Can be okay if the dataset is small.	
2. Incrementally load the data after the previous SCD is generated
	- We want the production data to be incremental so we don't process the all history each time it changes.

> Remember that all your pipelines don't have to be perfect

## Additional resources
> https://airbyte.com/data-engineering-resources/idempotency-in-data-pipelines 