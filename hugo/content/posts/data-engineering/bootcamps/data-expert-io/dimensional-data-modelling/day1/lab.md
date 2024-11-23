---
date: 2024-11-16T17:52:23+01:00
draft: false 
author: Gabriel LOPEZ
title: (DataExpert.io) Bootcamp - Day 1 - Lab
---

> **Goal :** Create a *cumulative table design*

## Problem overview 
We have a table containing the stats for the NBA players, there's one record for each player's season. 

```
postgres=# \d player_seasons;
```

Table `public.player_seasons`

| Column       | Type    | 
| ------------ | ------- | 
| player_name  | text    | 
| age          | integer | 
| height       | text    | 
| weight       | integer | 
| college      | text    | 
| country      | text    |
| draft_year   | text    |
| draft_round  | text    |
| draft_number | text    |
| gp           | real    |
| pts          | real    |
| reb          | real    |
| ast          | real    | 
| netrtg       | real    |  
| oreb_pct     | real    |   
| dreb_pct     | real    |    
| usg_pct      | real    |     
| ts_pct       | real    |      
| ast_pct      | real    |       
| season       | integer |        

**Indexes:**
`"player_seasons_pkey" PRIMARY KEY, btree (player_name, season)`

We have a temporal data problem with the table where joining the table with another would cause shuffling of the players records (same player statics won't be following each other) making **run-length encoding** compression less efficient
## Run-Length Encoding

> **Run-Length Encoding** is a simple data compression algorithm that encodes consecutive repeated data elements (runs) as a single value plus a count of its repetitions. 
 
Instead of storing the repeated data multiple times, it stores the data value and the number of times it appears consecutively.

We are going to transform the table to have one row per player with a column of arrays of the player seasons. 

## Cumulative table design

The cumulative design serves two distinct but complementary purposes:

1. **Join/GroupBy Optimization**:
    - By storing temporal data (seasons) together in arrays, we optimize for:
        - Fewer rows to join
        - Less data shuffling during grouping
        - Better data locality

2. **RLE Compression**: When we later explode/unnest the arrays, the data will naturally group temporal values together, making RLE more efficient.
 

## What things are part of a season and what things aren't ?
We want to store the temporal component in it's own data type. 

We create a `STRUCT` named `season_stats` with Postges :
```sql
CREATE TYPE season_stats AS (  
    season INTEGER,  
    gp INTEGER,  
    pts REAL,  
    reb REAL,  
    ast REAL  
)
```
We don't take all the season statistics in this struct as we won't need all of them.

## Creating the cumulative table
Then we create the cumulative table schema using our new `STRUCT` :
```sql
CREATE TABLE players (  
    player_name TEXT,  
    height TEXT,  
    college TEXT,  
    country TEXT,  
	draft_year TEXT,
    draft_round TEXT,  
    season_stats season_stats[],  
    current_season INTEGER,  
    PRIMARY KEY(player_name, current_season)  
)
```

We want to figure out what is the first year in the table is :
```sql
SELECT MIN(season) FROM player_seasons;
```

It is `1996`

```sql
WITH yesterday AS (  
    SELECT * FROM players  
    WHERE current_seasons = 1995  
),  
today AS (  
    SELECT * FROM player_seasons  
    WHERE season = 1996  
)  
  
SELECT * FROM today t FULL OUTER JOIN yesterday y  
    ON t.player_name = y.player_name
```

The request give us `<null>` values for the left side of the join `1995` (`yesterday`) as it doesn't exists.

Now we want to `COALESCE()` the non temporal values. 

```sql
WITH yesterday AS (  
    SELECT * FROM players  
    WHERE current_seasons = 1995  
),  
today AS (  
    SELECT * FROM player_seasons  
    WHERE season = 1996  
)  
SELECT 
	COALESCE(t.player_name, y.player_name) AS player_name,
	COALESCE(t.height, y.height) AS height,
	COALESCE(t.college, y.college) AS college,
	COALESCE(t.country, y.country) AS country,
	COALESCE(t.draft_year, y.draft_year) AS draft_year,
	COALESCE(t.draft_round, y.draft_round) AS draft_round,
	COALESCE(t.draft_number, y.draft_number) AS draft_number,
FROM today t FULL OUTER JOIN yesterday y  
    ON t.player_name = y.player_name
```

 **Purpose of COALESCE here**:
   - Handling data continuity between two time periods
   - It ensures we keep the non-temporal data when a player exists in either period
 
**What the query actually does**:
For each player: 
- If player exists in 'today' (1996): use today's data 
- If player only exists in 'yesterday' (1995): use yesterday's data 
- If player exists in both: use today's data (through COALESCE taking first non-NULL)`

```sql
SELECT * FROM player_seasons;  
  
DROP TABLE IF EXISTS players ;  
  
CREATE TYPE scoring_class AS ENUM('star', 'good', 'average', 'bad');  
  
CREATE TABLE players (  
    player_name TEXT,  
    height TEXT,  
    college TEXT,  
    country TEXT,  
    draft_year TEXT,  
    draft_round TEXT,  
    draft_number TEXT,  
    season_stats season_stats[],  
    scoring_class scoring_class,  
    years_since_last_season INTEGER,  
    current_season INTEGER,  
    PRIMARY KEY(player_name, current_season)  
);  
  
-- This is the SEED query for cumulation because year 1995 is going to be <null>,  
-- the FULL OUTER JOIN is just taking everything from today as yesterday doesn't exist.  
INSERT INTO players  
WITH yesterday AS (  
    SELECT * FROM players  
    WHERE current_season  = 2000  
),  
today AS (  
    SELECT * FROM player_seasons  
    WHERE season = 2001  
)  
  
SELECT  
    COALESCE(t.player_name, y.player_name) AS player_name,  
    COALESCE(t.height, y.height) AS height,  
    COALESCE(t.college, y.college) AS college,  
    COALESCE(t.country, y.country) AS country,  
    COALESCE(t.draft_year, y.draft_year) AS draft_year,  
    COALESCE(t.draft_round, y.draft_round) AS draft_round,  
    COALESCE(t.draft_number, y.draft_number) AS draft_number,  
    -- If yesterday is null we create the initial array  
    CASE WHEN y.season_stats IS NULL THEN  
        ARRAY[ROW(  
            t.season,  
            t.gp,  
            t.pts,  
            t.reb,  
            t.ast  
        )::season_stats]  
    -- If today is not null we create the new value by concatenating the array of previous values  
    -- with today's ones.    
    -- We don't want to keep adding values to the season_stats array if the player is retired 
    WHEN t.season IS NOT NULL THEN  
        y.season_stats || ARRAY[ROW(  
            t.season,  
            t.gp,  
            t.pts,  
            t.reb,  
            t.ast  
        )::season_stats]  
    -- Otherwise we carry the history forward without modifying it.  
    ELSE y.season_stats  
    END AS season_stats,  
    -- Determine the scoring class of the player for current season  
    CASE  
        WHEN t.season IS NOT NULL THEN  
        CASE WHEN t.pts > 20 THEN 'star'  
            WHEN t.pts >  15 THEN 'good'  
            WHEN t.pts > 10 THEN 'average'  
            ELSE 'bad'  
        END::scoring_class  
        ELSE y.scoring_class  
    END as scoring_class,  
    CASE WHEN t.season IS NOT NULL THEN 0  
        ELSE y.years_since_last_season + 1  
    END as years_since_last_season,  
    COALESCE(t.season, y.current_season + 1) as current_season  
FROM today t FULL OUTER JOIN yesterday y  
    ON t.player_name = y.player_name;  
  
-- No GROUP BY = very fast, everything happens in the map step it can be parrallelized
SELECT  
    player_name,  
    (season_stats[CARDINALITY(season_stats)]::season_stats).pts /  
    CASE WHEN (season_stats[1]::season_stats).pts = 0 THEN 1 ELSE ((season_stats[1]::season_stats).pts) END  
FROM players  
WHERE current_season = 2001  
AND scoring_class = 'star';  
  
-- Going back the original table  
WITH unnested AS (  
    SELECT player_name,  
        UNNEST(season_stats) AS season_stats  
    FROM players  
    WHERE current_season = 2001  
)  
  
SELECT player_name,  
       (season_stats::season_stats).*  
FROM unnested;  
  
-- Here we keep player stats (temporal attributes) sorted through the JOIN
-- We can apply RLE compression efficiently
```
