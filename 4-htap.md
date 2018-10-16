---
title: HTAP
summary: Tutorial for deploying TiDB on Google Cloud using Kubernetes.
category: operations
---

# HTAP

## Introduction

Using your previously deployed Kubernetes Cluster, we will complete the following steps:

- Recheck configuration
- Drop existing indexes
- Optimize Analytics Queries

## Recheck configuration

Establish a tunnel to TiDB:

	establish-port-forward
	
If you see `-bash: establish-port-forward: command not found`, reload the setup tutorial and make sure your environment is configured correctly.

## Drop existing indexes

In order to get the best learning experience, it's recommended to drop all of the existing indexes on the `trips` table:

```sql
ALTER TABLE trips DROP INDEX s;
ALTER TABLE trips DROP INDEX b;
ALTER TABLE trips DROP INDEX sb;
ALTER TABLE trips DROP INDEX bs;
SHOW CREATE TABLE trips\G
```

### Expected result

```
MySQL [bikeshare]> ALTER TABLE trips DROP INDEX s;
ALQuery OK, 0 rows affected (0.46 sec)

MySQL [bikeshare]> ALTER TABLE trips DROP INDEX b;
Query OK, 0 rows affected (0.40 sec)

MySQL [bikeshare]> ALTER TABLE trips DROP INDEX sb;
AQuery OK, 0 rows affected (0.38 sec)

MySQL [bikeshare]> ALTER TABLE trips DROP INDEX bs;
Query OK, 0 rows affected (0.35 sec)

MySQL [bikeshare]> SHOW CREATE TABLE trips\G
*************************** 1. row ***************************
       Table: trips
Create Table: CREATE TABLE `trips` (
  `trip_id` bigint(20) NOT NULL AUTO_INCREMENT,
  `duration` int(11) NOT NULL,
  `start_date` datetime DEFAULT NULL,
  `end_date` datetime DEFAULT NULL,
  `start_station_number` int(11) DEFAULT NULL,
  `start_station` varchar(255) DEFAULT NULL,
  `end_station_number` int(11) DEFAULT NULL,
  `end_station` varchar(255) DEFAULT NULL,
  `bike_number` varchar(255) DEFAULT NULL,
  `member_type` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`trip_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin AUTO_INCREMENT=3780001
1 row in set (0.00 sec)
```

## The simplest analytics query

We are going to start with a very simple analytics query:

```sql
SELECT COUNT(*) c FROM trips;
```

### Expected result

```
MySQL [bikeshare]> SELECT COUNT(*) c FROM trips;
+---------+
| c       |
+---------+
| 3757777 |
+---------+
1 row in set (1.08 sec)
```

In `EXPLAIN`:

```sql
EXPLAIN SELECT COUNT(*) c FROM trips;
```

### Expected result 
```
MySQL [bikeshare]> EXPLAIN SELECT COUNT(*) c FROM trips;
+------------------------+------------+------+--------------------------------------------------+
| id                     | count      | task | operator info                                    |
+------------------------+------------+------+--------------------------------------------------+
| StreamAgg_16           | 1.00       | root | funcs:count(col_0)                               |
| └─TableReader_17       | 1.00       | root | data:StreamAgg_8                                 |
|   └─StreamAgg_8        | 1.00       | cop  | funcs:count(1)                                   |
|     └─TableScan_15     | 3757777.00 | cop  | table:trips, range:[-inf,+inf], keep order:false |
+------------------------+------------+------+--------------------------------------------------+
4 rows in set (0.01 sec)
```

Let’s explain what's happening here:

* A tablescan operation is reading each TiKV Region in parallel. There is no WHERE clause, so there is a tablescan between `-inf` and `+inf` (all rows in the table).

* The coprocessor has the built-in function `count`, so that means that the data on each of the Regions is aggregated as part of `StreamAgg_8` down to an estimated 1.00 rows. It is the 1.00 rows that is sent back across the network to TiDB as part of TableReader_17.

* Because there were multiple Regions that were read, these all need to be aggregated to produce the final result - a single row count to be returned to the client.

## Finding the most popular station

In this query, we are finding which stations are the most popular:

```sql
SELECT COUNT(*) c, start_station_number FROM trips GROUP BY start_station_number ORDER BY c DESC LIMIT 10;
```

### Expected result
```
MySQL [bikeshare]> SELECT COUNT(*) c, start_station_number FROM trips GROUP BY start_station_number ORDER BY c DESC LIMIT 10;
+-------+----------------------+
| c     | start_station_number |
+-------+----------------------+
| 70062 |                31623 |
| 65884 |                31258 |
| 59259 |                31247 |
| 46702 |                31200 |
| 43305 |                31201 |
| 42525 |                31249 |
| 42406 |                31248 |
| 40659 |                31289 |
| 37751 |                31288 |
| 33159 |                31101 |
+-------+----------------------+
10 rows in set (3.65 sec)
```

In EXPLAIN:

```sql
EXPLAIN SELECT COUNT(*) c, start_station_number FROM trips GROUP BY start_station_number ORDER BY c DESC LIMIT 10;
```

### Expected result

```
MySQL [bikeshare]> EXPLAIN SELECT COUNT(*) c, start_station_number FROM trips GROUP BY start_station_number ORDER BY c DESC LIMIT 10;
+--------------------------+------------+------+---------------------------------------------------------------------------------------------------------------+
| id                       | count      | task | operator info                                                                                                 |
+--------------------------+------------+------+---------------------------------------------------------------------------------------------------------------+
| TopN_10                  | 10.00      | root | 2_col_0:desc, offset:0, count:10                                                                              |
| └─HashAgg_18             | 487.00     | root | group by:col_2, funcs:count(col_0), firstrow(col_1)                                                           |
|   └─TableReader_19       | 487.00     | root | data:HashAgg_14                                                                                               |
|     └─HashAgg_14         | 487.00     | cop  | group by:bikeshare.trips.start_station_number, funcs:count(1), firstrow(bikeshare.trips.start_station_number) |
|       └─TableScan_17     | 3757777.00 | cop  | table:trips, range:[-inf,+inf], keep order:false                                                              |
+--------------------------+------------+------+---------------------------------------------------------------------------------------------------------------+
5 rows in set (0.00 sec)
```

So let's start by describing what's happening:

* We have a tablescan again, because there is no WHERE clause or ability to filter rows.

* The tablescan operation feeds into a HashAggregation, where rows inserted into a hash table to produce the grouping of how many rows there are per start_station_number. The HashAggregation is required because the table scan is not ordered, and we need to keep some sort of buffer of running totals while performing the table scan.

* The grouped results in the hash table are then sent to TiDB (as TableReader_19) where they need to be grouped again in another hash table to combine all of the results from other TiKV servers.

* A final TopN can be applied to retrieve the 10 highest values.

So for those from a MySQL background, the hash aggregation is an improvement over execution strategies that are possible in MySQL. There are temporary tables which can be used to buffer intermediate results, but these are less efficient than hash tables.

But this query can be improved, and the hash aggregation step can be removed. To do so, we need to add an index.  This operation will take approximately 10 minutes:

```sql
ALTER TABLE trips ADD INDEX (start_station_number);
EXPLAIN SELECT COUNT(*) c, start_station_number FROM trips GROUP BY start_station_number ORDER BY c DESC LIMIT 10;
```

### Expected result

```
MySQL [bikeshare]> ALTER TABLE trips ADD INDEX (start_station_number);
Query OK, 0 rows affected (8 min 55.81 sec)

MySQL [bikeshare]> EXPLAIN SELECT COUNT(*) c, start_station_number FROM trips GROUP BY start_station_number ORDER BY c DESC LIMIT 10;
+--------------------------+------------+------+---------------------------------------------------------------------------------------------------------------+
| id                       | count      | task | operator info                                                                                                 |
+--------------------------+------------+------+---------------------------------------------------------------------------------------------------------------+
| TopN_10                  | 10.00      | root | 2_col_0:desc, offset:0, count:10                                                                              |
| └─StreamAgg_26           | 474.25     | root | group by:col_2, funcs:count(col_0), firstrow(col_1)                                                           |
|   └─IndexReader_27       | 474.25     | root | index:StreamAgg_17                                                                                            |
|     └─StreamAgg_17       | 474.25     | cop  | group by:bikeshare.trips.start_station_number, funcs:count(1), firstrow(bikeshare.trips.start_station_number) |
|       └─IndexScan_25     | 1317777.00 | cop  | table:trips, index:start_station_number, range:[NULL,+inf], keep order:true                                   |
+--------------------------+------------+------+---------------------------------------------------------------------------------------------------------------+
5 rows in set (0.11 sec)
```

* Here the table scan has changed to an index scan on `start_station_number`. Even though this is an index scan (the full range of the index is examined), it is already more efficient because the index is less wide than the table.
* A second feature of the index, is that because the index is ordered, the parent of the index scan is now a stream aggregation and not a hash aggregation. This makes sense: because all the rows are ordered, it can just count the number of rows while reading the records without having to buffer an intermediate result.
* The results of the stream aggregator are then ordered themselves, so they hand over rows to an IndexReader in TiDB, which is then Stream Aggregated to combine the results from the other TiKV servers.
* Finally, a Top N is applied to reduce the result to only the Top 10 rows.

By switching from hash aggregation to stream aggregation, and scanning the less-wide index, query execution time changes from 3.65 seconds to 2.60 seconds. Not a bad improvement!
