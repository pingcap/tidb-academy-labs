---
title: Restore from Accidental Delete
summary: Tutorial for deploying TiDB on Google Cloud using Kubernetes.
category: operations
---

# Restore from Accidental Delete

## Introduction

Using your previously deployed Kubernetes Cluster, we will complete the following steps:

- Accidentally delete data
- Use `tidb_snapshot` to restore data without backup
- Verify data is restored

## Recheck configuration

Establish a tunnel to TiDB:

	establish-port-forward
	
If you see `-bash: establish-port-forward: command not found`, reload the setup tutorial and make sure your environment is configured correctly.

## Perform the accident

In this case, we will be deleting data on a full day. Write down the time returned from `NOW()`, and you will need it for the next step: 

```sql
use bikeshare;
SELECT NOW();
SELECT count(*) FROM trips WHERE start_date BETWEEN '2017-01-01' AND '2017-01-02';
```

### Expected result
```sql
MySQL [(none)]> use bikeshare;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MySQL [bikeshare]> SELECT NOW();
+---------------------+
| NOW()               |
+---------------------+
| 2018-10-12 02:03:26 |
+---------------------+
1 row in set (0.01 sec)

MySQL [bikeshare]> SELECT count(*) FROM trips WHERE 
    ->     start_date BETWEEN '2017-01-01' 
    ->     AND '2017-01-02';
+----------+
| count(*) |
+----------+
|     4063 |
+----------+
1 row in set (3.25 sec)
```

Delete the data and confirm it has been removed:

```sql
DELETE FROM trips WHERE start_date BETWEEN '2017-01-01' AND '2017-01-02';
SELECT count(*) FROM trips WHERE start_date BETWEEN '2017-01-01' AND '2017-01-02';
```

### Expected result

```sql
MySQL [bikeshare]> DELETE FROM trips WHERE start_date 
    ->     BETWEEN '2017-01-01' AND 
    ->     '2017-01-02';

Query OK, 4063 rows affected (3.95 sec)

MySQL [bikeshare]> SELECT count(*) FROM trips WHERE 
    ->     start_date BETWEEN '2017-01-01' 
    ->     AND '2017-01-02';
+----------+
| count(*) |
+----------+
|        0 |
+----------+
1 row in set (3.49 sec)
```

## Set the time to the past

Using [steps described in the TiDB manual](https://pingcap.com/docs/op-guide/history-read/), set the `@@tidb_snapshot` to be the time returned before data was deleted:

For example:

```sql
SET @@tidb_snapshot="2018-09-10 15:51:00";
```

You should now be able to read the deleted data:

```sql
SELECT count(*) FROM trips WHERE start_date BETWEEN '2017-01-01' AND '2017-01-02';
```

### Expected result

```sql
MySQL [bikeshare]> SET @@tidb_snapshot="2018-10-12 02:03:26";
Query OK, 0 rows affected (0.01 sec)

MySQL [bikeshare]> SELECT count(*) FROM trips WHERE start_date BETWEEN '2017-01-01' AND '2017-01-02';
+----------+
| count(*) |
+----------+
|     4063 |
+----------+
1 row in set (2.86 sec)
```

## Restore old rows

Because `tidb_snapshot` also puts your connection in a read only mode, we need to export the rows to `csv` and then reimport them in the current time. The easiest way to do this is as follows:

```
mysql bikeshare -BNe "SET @@tidb_snapshot='YOUR_TIMESTAMP_HERE'; SELECT * FROM trips WHERE start_date BETWEEN '2017-01-01' AND '2017-01-02';" > missing_rows.txt
cat missing_rows.txt | wc -l
mysql bikeshare -e "LOAD DATA LOCAL INFILE 'missing_rows.txt' INTO TABLE trips"
```

Please make sure you set `YOUR_TIMESTAMP_HERE` to the timestamp you used in the previous step.

### Expected result

```
tocker@cloudshell:~/tidb-academy (morgan-test-213703)$ mysql bikeshare -BNe "SET @@tidb_snapshot='2018-10-12 02:03:26'; SELECT * FROM trips WHERE start_date BETWEEN '2017-01-01' AND '2017-01-02';" > missing_rows.txt

tocker@cloudshell:~ (morgan-test-213703)$ cat missing_rows.txt | wc -l
4063
tocker@cloudshell:~ (morgan-test-213703)$ mysql bikeshare -e "LOAD DATA LOCAL INFILE 'missing_rows.txt' INTO TABLE trips"

```

## Verifying data is restored

Back in the MySQL client:

```sql
use bikeshare;
SELECT count(*) FROM trips WHERE start_date BETWEEN '2017-01-01' AND '2017-01-02';
```

### Expected result

```sql
MySQL [bikeshare]> SELECT count(*) FROM trips WHERE start_date BETWEEN '2017-01-01' AND '2017-01-02';
+----------+
| count(*) |
+----------+
|     4063 |
+----------+
1 row in set (2.87 sec)
```
