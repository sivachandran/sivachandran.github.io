---
layout: post
title: "Optimising Time Window Queries with Postgres Timestamp Range Data Types"
date: 2019-05-04 10:17:00 +0530
tags: [postgres, database, sql]
---
This post explains how we were able to improve a database query performance by replacing two individual `timestamptz` columns with single Postgres's `tstzrange` range column.

It all began when our operation team complaint that a particular report is taking really lots of time to generate. Initial analysis revealed the report spent most of the time in executing a particular query in our Postgres database. Though the query is joining(inner) three tables, the join and where clause are straightforward. We also confirmed indexes exist for the columns involved in join and where clause. So we fired up [pgHero](https://github.com/ankane/pghero) and generated the execution plan for the query. Loading the execution plan into [PEV](https://tatiyants.com/pev) indicated that the where clause on timestamp columns are not using the indexes and had to scan all rows.

To understand the problem deeper let us consider the following pseudo table models.

### *theatres* table

| id | name |
|----|------|
| 1000 | Theatre 1 |
| 1001 | Theatre 2 |


### *screens* table

| id | name |
|----|------|
| 2000 | Screen 1 |
| 2001 | Screen 2 |

### *licenses* table

| id | screen_id | theatre_id | valid_from | valid_till |
|----|-----------|------------|------------|------------|
| 3000 | 2000 | 1000 | 2019-04-01T00:00:00+0000 | 2019-05-15T23:59:59+0000 |
| 3001 | 2001 | 1001 | 2019-04-01T00:00:00+0000 | 2019-05-15T23:59:59+0000 |
| 3002 | 2000 | 1000 | 2019-05-01T00:00:00+0000 | 2019-05-31T23:59:59+0000 |
| 3003 | 2001 | 1001 | 2019-05-01T00:00:00+0000 | 2019-05-31T23:59:59+0000 |

The report is about listing all theatre, screen pairs having any valid movie license within the specified date/time range. The offending query was written like below:

```sql
SELECT DISTINCT t.id AS theatre_id, s.id AS screen_id
FROM theatres t
INNER JOIN licenses l ON l.theatre_id = t.id
INNER JOIN screens s ON s.id = l.screen_id
WHERE (l.valid_from, l.valid_till) OVERLAPS ('2019-04-01T00:00:00+0000', '2019-05-31T23:59:59+0000')
```

The above query took **~450ms** to select **140** rows on tables **theatres**, **screens** and **licenses** with row count of **37339**, **147850**, **350693** respectively. The execution plan indicated the query uses indexes for the various `id` columns but didn't use the indexes for `valid_from` and `valid_till` column.

Suspecting the timestamp indexes, we rewrote the query like below to check the usage of operator `OVERLAPS` prevents the query using the indexes.

```sql
SELECT DISTINCT t.id AS theatre_id, s.id AS screen_id
FROM theatres t
INNER JOIN licenses l ON l.theatre_id = t.id
INNER JOIN screens s ON s.id = l.screen_id
WHERE (l.valid_from >= '2019-04-01T00:00:00+0000' AND l.valid_from <= '2019-05-31T23:59:59+0000')
    OR (l.valid_till >= '2019-04-01T00:00:00+0000' AND l.valid_till <= '2019-05-31T23:59:59+0000')
     OR (l.valid_from <= '2019-04-01T00:00:00+0000' AND l.valid_till >= '2019-05-31T23:59:59+0000')
```

To our surprise the query is fast, it took just **~180ms* to select the same **140** rows from the same dataset. The execution plan showed it uses the timestamp indexes. We couldn't understand why the modified query is able to use indexes and faster, but the original query with `OVERLAPS` operator couldn't and slower. We did a little search on the Internet but we couldn't gather much info. But thinking about it aloud we realized rationale. The operator `OVERLAPS` checks whether two timestamp ranges overlaps or not. But the indexes are created for the individual timestamp columns. It is understandable that the operator can't take advantage of the indexes as it requires a single timestamp range comprising both the start and end timestamps, i.e. the `valid_from` and `valid_till`. So the query had to scan all rows to construct the timestamp range, combined value of `valid_from` and `valid_till`, to check against the specified timestamp range.

As we realized the separate indexes doesn't help time window overlap condition, we were set to find out is there a way to create an index for time window/timestamp ranges. We found that Postgres has builtin timestamp range data types `tsrange` (without time zone info) and `tstzrange` (with time zone info). So we replaced the two columns `valid_from` and `valid_till` with single column `validity` of type `tstzrange`. We also created a compatible index, GiST, for the column data. While migrating the data from existing columns, we also found invalid data of kind `valid_till` being less than `valid_from`. The `tstzrange` type's builtin validation caught these invalid data and we were able to fix them. 

### *licenses* table with column *validity*

| id | screen_id | theatre_id | validity |
|----|-----------|------------|-----------|
| 3000 | 2000 | 1000 | ["2019-04-01T00:00:00+0000","2019-05-15T23:59:59+0000"] |
| 3001 | 2001 | 1001 | ["2019-04-01T00:00:00+0000","2019-05-15T23:59:59+0000"] |
| 3002 | 2000 | 1000 | ["2019-05-01T00:00:00+0000","2019-05-31T23:59:59+0000"] |
| 3003 | 2001 | 1001 | ["2019-05-01T00:00:00+0000","2019-05-31T23:59:59+0000"] |

We rewrote the query like below to achieve the result with the new `validity` column of type `tstzrange`.

```sql
SELECT DISTINCT t.id AS theatre_id, s.id AS screen_id
FROM theatres t
INNER JOIN licenses l ON l.theatre_id = t.id
INNER JOIN screens s ON s.id = l.screen_id
WHERE l.validity && tstzrange('["2019-04-01T00:00:00+0000", "2019-05-31T23:59:59+0000"]')
```

Now the query is as fast as the previous modified version, took **~180ms** to produce same **140** rows from same the dataset. But the usage of `validity` column with `tstzrange` type's operator `&&` made the query succinct. Additionally `tstzrange` type also brought in automatic validation for the timestamp range data. 

We were happy that we not only made the query fast but also learned something new in Postgres. We hope this experience and learning of ours will be useful to you.

### References
[Postgres Range Types](https://www.postgresql.org/docs/11/rangetypes.html)

*This post is originally published [here](https://engineering.qubecinema.com/2019/05/04/time-window-queries-with-postgres-timestamp-range.html)*.
