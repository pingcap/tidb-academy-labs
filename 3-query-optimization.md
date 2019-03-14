---
title: Query Optimization
summary: Tutorial for deploying TiDB on Google Cloud using Kubernetes.
category: operations
---

# Query Optimization

## Introduction

Using your previously deployed Kubernetes Cluster, we will complete the following step:

- Optimize Queries

## Recheck configuration

Establish a tunnel to TiDB:

  establish-port-forward
	
If you see `-bash: establish-port-forward: command not found`, reload the setup tutorial and make sure your environment is configured correctly.

## Example query

The first query we are going to optimize is to find all the trips that a particular bike took one day in December 2017:

```sql
use bikeshare;
SELECT * FROM trips WHERE bike_number = 'W22041' and DATE(start_date) = '2017-12-01'\G
```

### Expected result

```
SELECT * FROM trips WHERE bike_number = 'W22041' and DATE(start_date) = '2017-12-01'\G
*************************** 1. row ***************************
             trip_id: 3582371
            duration: 82
          start_date: 2017-12-01 09:11:14
            end_date: 2017-12-01 09:12:36
start_station_number: 31030
       start_station: Lee Hwy & N Adams St
  end_station_number: 31030
         end_station: Lee Hwy & N Adams St
         bike_number: W22041
         member_type: Member
*************************** 2. row ***************************
             trip_id: 3582412
            duration: 499
          start_date: 2017-12-01 09:16:08
            end_date: 2017-12-01 09:24:28
start_station_number: 31030
       start_station: Lee Hwy & N Adams St
  end_station_number: 31014
         end_station: Lynn & 19th St North
         bike_number: W22041
         member_type: Casual
2 rows in set (6.35 sec)
```

If you run this query, it takes approximately 6 seconds to execute. How can we improve the performance?

## Using EXPLAIN

The first step is to run the query through `EXPLAIN`. This does not execute the query, but shows the execution plan TiDB has chosen for execution:

```sql
EXPLAIN SELECT * FROM trips WHERE bike_number = 'W22041' and DATE(start_date) = '2017-12-01';
```

### Expected result

```
MySQL [bikeshare]> EXPLAIN SELECT * FROM trips WHERE bike_number = 'W22041' and DATE(start_date) = '2017-12-01';
+-----------------------+------------+------+------------------------------------------------------------------+
| id                    | count      | task | operator info                                                    |
+-----------------------+------------+------+------------------------------------------------------------------+
| Selection_5           | 16.00      | root | eq(date(bikeshare.trips.start_date), 2017-12-01 00:00:00.000000) |
| └─TableReader_8       | 20.00      | root | data:Selection_7                                                 |
|   └─Selection_7       | 20.00      | cop  | eq(bikeshare.trips.bike_number, "W22041")                        |
|     └─TableScan_6     | 3757777.00 | cop  | table:trips, range:[-inf,+inf], keep order:false                 |
+-----------------------+------------+------+------------------------------------------------------------------+
4 rows in set (0.00 sec)
```

We can see from `TableScan_6` in the operator info that the trips table needs to be scanned from `-inf` to `+inf`: there is no index available to filter on either `bike_number = 'W22041'` or `DATE(start_date) = '2017-12-01'`.

## Evaluating index choices

With our query, we have a number of index choices available:

* An index on `s (start_date)`
* An index on `b (bike_number)`
* A composite index on `sb (start_date, bike_number)`
* A composite index on `bs (bike_number, start_date)`

So what is the best choice? While you would not do this in production, one of the goals of the optimizer is to select the best index, so for educational purposes, let's add all the indexes.

## Index `s`

Add the index `s`, which will take approximately 7 minutes. When it is complete, we run `EXPLAIN` again:

```sql
ALTER TABLE trips ADD INDEX s (start_date);
EXPLAIN SELECT * FROM trips WHERE bike_number = 'W22041' and DATE(start_date) = '2017-12-01';
```

### Expected result

```
ALTER TABLE trips ADD INDEX s (start_date);
MySQL [bikeshare]> EXPLAIN SELECT * FROM trips WHERE bike_number = 'W22041' and DATE(start_date) = '2017-12-01';
+-----------------------+------------+------+------------------------------------------------------------------+
| id                    | count      | task | operator info                                                    |
+-----------------------+------------+------+------------------------------------------------------------------+
| Selection_5           | 16.00      | root | eq(date(bikeshare.trips.start_date), 2017-12-01 00:00:00.000000) |
| └─TableReader_8       | 20.00      | root | data:Selection_7                                                 |
|   └─Selection_7       | 20.00      | cop  | eq(bikeshare.trips.bike_number, "W22041")                        |
|     └─TableScan_6     | 3757777.00 | cop  | table:trips, range:[-inf,+inf], keep order:false                 |
+-----------------------+------------+------+------------------------------------------------------------------+
4 rows in set (0.00 sec)
```

After we run `EXPLAIN` again, we can see that nothing has changed! Despite the index being added, the `TableScan_6` still shows a range of rows between `-inf` and `+inf`. Why is this?

The first thing to point out, is that it is *not* because `DATE(start_date) = '2017-12-01'` appears second in the WHERE clause:

```sql
EXPLAIN SELECT * FROM trips WHERE DATE(start_date) = '2017-12-01' and bike_number = 'W22041';
```

### Expected result

```
MySQL [bikeshare]> EXPLAIN SELECT * FROM trips WHERE DATE(start_date) = '2017-12-01' and bike_number = 'W22041';
+-----------------------+------------+------+------------------------------------------------------------------+
| id                    | count      | task | operator info                                                    |
+-----------------------+------------+------+------------------------------------------------------------------+
| Selection_5           | 16.00      | root | eq(date(bikeshare.trips.start_date), 2017-12-01 00:00:00.000000) |
| └─TableReader_8       | 20.00      | root | data:Selection_7                                                 |
|   └─Selection_7       | 20.00      | cop  | eq(bikeshare.trips.bike_number, "W22041")                        |
|     └─TableScan_6     | 3757777.00 | cop  | table:trips, range:[-inf,+inf], keep order:false                 |
+-----------------------+------------+------+------------------------------------------------------------------+
4 rows in set (0.01 sec)
```

The reason is because `DATE(start_date) = '2017-12-01'` is not _sargable_ (a portmanteau of search argument-able). In most databases (TiDB included), a function on a column prevents the use of an index. If we rewrite the statement to the following form, we can see that an index will be used:

```sql
EXPLAIN SELECT * FROM trips WHERE bike_number='W22041' and start_date BETWEEN '2017-12-01 00:00:00' AND '2017-12-01 23:59:59';
```

### Expected result

```
MySQL [bikeshare]> EXPLAIN SELECT * FROM trips WHERE bike_number='W22041' and start_date BETWEEN '2017-12-01 00:00:00' AND '2017-12-01 23:59:59';
+---------------------+---------+------+--------------------------------------------------------------------------------------------------+
| id                  | count   | task | operator info                                                                                    |
+---------------------+---------+------+--------------------------------------------------------------------------------------------------+
| IndexLookUp_11      | 0.03    | root |                                                                                                  |
| ├─IndexScan_8       | 3835.72 | cop  | table:trips, index:start_date, range:[2017-12-01 00:00:00,2017-12-01 23:59:59], keep order:false |
| └─Selection_10      | 0.03    | cop  | eq(bikeshare.trips.bike_number, "W22041")                                                        |
|   └─TableScan_9     | 3835.72 | cop  | table:trips, keep order:false                                                                    |
+---------------------+---------+------+--------------------------------------------------------------------------------------------------+
4 rows in set (0.01 sec)
```

The execution plan has now changed! `IndexScan_8` is used to filter rows between `range:[2017-12-01 00:00:00,2017-12-01 23:59:59]`. The index returns an estimated 3835.72 rows, each need to read a row in the table and check whether the bike_number matches `W22041`. 

Running the query shows that the time has reduced to 0.06 seconds. Not bad!

## Index `b`

The next index to evaluate is on the bike_number itself. This step will take approximately 7 minutes:

```sql
ALTER TABLE trips ADD INDEX b (bike_number);
EXPLAIN SELECT * FROM trips WHERE bike_number='W22041' and start_date BETWEEN '2017-12-01 00:00:00' AND '2017-12-01 23:59:59';
```

### Expected result

```
ALTER TABLE trips ADD INDEX b (bike_number);
MySQL [bikeshare]> EXPLAIN SELECT * FROM trips where bike_number='W22041' and start_date BETWEEN '2017-12-01 00:00:00' AND '2017-12-01 23:59:59';
+----------------------+-------+------+------------------------------------------------------------------------------------------------------------------------+
| id                   | count | task | operator info                                                                                                          |
+----------------------+-------+------+------------------------------------------------------------------------------------------------------------------------+
| IndexLookUp_15       | 0.03  | root |                                                                                                                        |
| ├─IndexScan_12       | 20.00 | cop  | table:trips, index:bike_number, range:["W22041","W22041"], keep order:false                                            |
| └─Selection_14       | 0.03  | cop  | ge(bikeshare.trips.start_date, 2017-12-01 00:00:00.000000), le(bikeshare.trips.start_date, 2017-12-01 23:59:59.000000) |
|   └─TableScan_13     | 20.00 | cop  | table:trips, keep order:false                                                                                          |
+----------------------+-------+------+------------------------------------------------------------------------------------------------------------------------+
4 rows in set (0.01 sec)
```

Here we can see that `EXPLAIN` shows that the bike_number index is preferred over the index on start_date. If we run the actual query, we can also see that the execution time has reduced from 0.06 seconds to 0.03 seconds.

How did the query optimizer know that `b` was a better index? We can see that in `EXPLAIN`. The statistics that TiDB keeps on indexes estimated that there are 20 rows that match `bike_number = 'W22041'`. An index that matches fewer rows means less rows are needed for the `TableScan_13` operation.

## Index `sb`

If the index on bike_number was able to reduce the number of rows scanned, then can an index on the composite of start_date and bike_number further reduce filtering? Is there a preference between `(start_date, bike_number)` and `(bike_number, start_date)`? Let's check.

This step will take approximately 7 minutes:

```sql
ALTER TABLE trips ADD INDEX sb (start_date, bike_number);
EXPLAIN SELECT * FROM trips WHERE bike_number='W22041' and start_date BETWEEN '2017-12-01 00:00:00' AND '2017-12-01 23:59:59';
```

### Expected result

```
ALTER TABLE trips ADD INDEX sb (start_date, bike_number);
MySQL [bikeshare]> EXPLAIN SELECT * FROM trips WHERE bike_number='W22041' and start_date BETWEEN '2017-12-01 00:00:00' AND '2017-12-01 23:59:59';
+----------------------+-------+------+------------------------------------------------------------------------------------------------------------------------+
| id                   | count | task | operator info                                                                                                          |
+----------------------+-------+------+------------------------------------------------------------------------------------------------------------------------+
| IndexLookUp_15       | 0.03  | root |                                                                                                                        |
| ├─IndexScan_12       | 20.00 | cop  | table:trips, index:bike_number, range:["W22041","W22041"], keep order:false                                            |
| └─Selection_14       | 0.03  | cop  | ge(bikeshare.trips.start_date, 2017-12-01 00:00:00.000000), le(bikeshare.trips.start_date, 2017-12-01 23:59:59.000000) |
|   └─TableScan_13     | 20.00 | cop  | table:trips, keep order:false                                                                                          |
+----------------------+-------+------+------------------------------------------------------------------------------------------------------------------------+
4 rows in set (0.01 sec)
```

We can see from EXPLAIN that the index on the single column `bike_number` is preferred over a composite index. This is another common problem with index usage: because `start_date` is a range, only the first part of the composite index can be used.

## Index `bs`

If we change the order to be _const_ then range, the full index can be used. This step will take approximately 7 minutes:

```sql
ALTER TABLE trips ADD INDEX bs (bike_number, start_date);
EXPLAIN SELECT * FROM trips WHERE bike_number='W22041' and start_date BETWEEN '2017-12-01 00:00:00' AND '2017-12-01 23:59:59';
```

### Expected result

```
ALTER TABLE trips ADD INDEX bs (bike_number, start_date);
MySQL [bikeshare]> EXPLAIN SELECT * FROM trips WHERE bike_number='W22041' and start_date BETWEEN '2017-12-01 00:00:00' AND '2017-12-01 23:59:59';
+-------------------+-------+------+---------------------------------------------------------------------------------------------------------------------------------+
| id                | count | task | operator info                                                                                                                   |
+-------------------+-------+------+---------------------------------------------------------------------------------------------------------------------------------+
| IndexLookUp_10    | 0.02  | root |                                                                                                                                 |
| ├─IndexScan_8     | 0.02  | cop  | table:trips, index:bike_number, start_date, range:["W22041" 2017-12-01 00:00:00,"W22041" 2017-12-01 23:59:59], keep order:false |
| └─TableScan_9     | 0.02  | cop  | table:trips, keep order:false                                                                                                   |
+-------------------+-------+------+---------------------------------------------------------------------------------------------------------------------------------+
3 rows in set (0.03 sec)
```

## Conclusion

In this lab, we practiced adding indexes to improve the performance of a simple query. We will revisit `EXPLAIN` again later in this course when we take a look at analytics queries under HTAP.
