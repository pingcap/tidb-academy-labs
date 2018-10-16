---
title: Recover from Disaster
summary: Tutorial for deploying TiDB on Google Cloud using Kubernetes.
category: operations
---

# Recover from Disaster

## Introduction

Using your previously deployed Kubernetes Cluster, we will complete the following steps:

- Download and install mydumper
- Perform a full backup
- Simulate failure
- Restore from backup
- Confirm data is restored

## Recheck configuration

Establish a tunnel to TiDB:

	establish-port-forward
	
If you see `-bash: establish-port-forward: command not found`, reload the setup tutorial and make sure your environment is configured correctly.

## Download and install mydumper

We are going to use `mydumper` to perform a full backup of our TiDB platform. If you are familiar with `mysqldump` for MySQL, `mydumper` works similarly, but with a number of advantages such as parallel copy.

Download `mydumper` and `loader` and install in your `~/bin` directory:

```
# Download the tool package.
wget http://download.pingcap.org/tidb-enterprise-tools-latest-linux-amd64.tar.gz
wget http://download.pingcap.org/tidb-enterprise-tools-latest-linux-amd64.sha256
	
# Check the file integrity. If the result is OK, the file is correct.
sha256sum -c tidb-enterprise-tools-latest-linux-amd64.sha256
	
# Extract the package.
tar -xzf tidb-enterprise-tools-latest-linux-amd64.tar.gz
ls tidb-enterprise-tools-latest-linux-amd64/bin
cp tidb-enterprise-tools-latest-linux-amd64/bin/{mydumper,loader} ~/bin
```

## Perform a full backup

Using close to the default options available, we can perform a backup with:

```
mydumper -P 4000 --outputdir=backup
ls -lh backup
```

This process will take approximately 1 minute.

## Simulate failure

The next step is switch back to your `mysql` client and simulate a catastrophic failure in TiDB:

```sql
DROP DATABASE bikeshare;
```

### Expected result

```sql
MySQL [(none)]> DROP DATABASE bikeshare;
Query OK, 0 rows affected (0.35 sec)
```

## Restore from backup

To restore the data, we can use the parallel restore utility called `loader`:

```
cd backup
loader
```

This process will take approximately **15 minutes**.

## Confirm data has been restored

Once `loader` has finished, log back into MySQL and confirm that data has been restored:

```
use bikeshare;
SELECT COUNT(*) FROM trips;
```
	
You should see `3757781` rows in the table.
