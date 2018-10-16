---
title: Online DDL
summary: Tutorial for deploying TiDB on Google Cloud using Kubernetes.
category: operations
---

# Online DDL

## Introduction

Using your previously deployed Kubernetes Cluster, we will complete the following steps:

- Start a transaction
- In a second session add a column to the table
- Update more data
- Commit

## Recheck configuration

Establish a tunnel to TiDB:

	establish-port-forward
	
If you see `-bash: establish-port-forward: command not found`, reload the setup tutorial and make sure your environment is configured correctly.

## Setup second shell session

This exercise requires two active sessions to MySQL. I recommend using two cloud shell terminals, but if you are comfortable with using `screen`, you can do so as well.

![alt text](https://github.com/pingcap/tidb-academy-labs/raw/master/mysql_dbas/ddl/add-cloud-shell.png "Add Cloud Shell")

In both sessions, start `mysql` and switch to the `bikeshare` schema:

```
use bikeshake;
```

## Main exercise

Start a transaction and modify one row. Do not commit the change yet:

```sql
# session1
START TRANSACTION;
SELECT * FROM trips WHERE trip_id = 1000\G
UPDATE trips SET start_station='Nowhere' WHERE trip_id = 1000;
SELECT * FROM trips WHERE trip_id = 1000\G
```

Change the table. This should complete quickly. Afterwards, you can see the row, still without changes committed from session #1:

```sql
# session2
SELECT * FROM trips WHERE trip_id = 1000\G
ALTER TABLE trips ADD city varchar(255);
SELECT * FROM trips WHERE trip_id = 1000\G
```

Session #1 does not show the column yet. And it can still make changes:

```sql
# session1
SELECT * FROM trips WHERE trip_id = 1000\G
UPDATE trips SET start_station='Nowhere' WHERE trip_id = 1001;
```

Commit the transaction, and then select from the table again:

```sql
# session1
SELECT * FROM trips WHERE trip_id = 1000\G
COMMIT;
SELECT * FROM trips WHERE trip_id = 1000\G
```

After the commit the column should appear visible.
