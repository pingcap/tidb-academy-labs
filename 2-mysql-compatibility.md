---
title: Load Sample Data
summary: Tutorial for deploying TiDB on Google Cloud using Kubernetes.
category: operations
---

# Load Sample Data

## Introduction

This document introduces how to deploy TiDB on Google Cloud using your previously deployed Kubernetes Cluster. It includes the following steps:

- Download sample data (in CSV files)
- Create Table Structure in TiDB
- Import sample data into TiDB

## Recheck configuration

Establish a tunnel to TiDB:

	establish-port-forward
	
If you see `-bash: establish-port-forward: command not found`, reload the setup tutorial and make sure your environment is configured correctly.

## Download bikeshare sample data

From [the TiDB manual](https://pingcap.com/docs/bikeshare-example-database/), download the sample data file for year 2017:

	mkdir -p bikeshare-data && cd bikeshare-data &&
	wget https://s3.amazonaws.com/capitalbikeshare-data/2017-capitalbikeshare-tripdata.zip &&
	unzip 2017-capitalbikeshare-tripdata.zip

Downloading and extracting requires approximately 600MB of disk space.

**Note:** In the previous lab, the default recommendation had you setup a 3-node Kubernetes cluster with network attached disks, so that you could safely operate within Google Cloud's free tier. As a result, we are only loading one year's worth of sample data to account for a much slower TiDB Platform.

## Create a table

Create the bikeshare trips table in the MySQL shell. We will import the sample data in the next step:

```
CREATE DATABASE bikeshare;
USE bikeshare;

CREATE TABLE trips (
 trip_id bigint NOT NULL PRIMARY KEY auto_increment,
 duration integer not null,
 start_date datetime,
 end_date datetime,
 start_station_number integer,
 start_station varchar(255),
 end_station_number integer,
 end_station varchar(255),
 bike_number varchar(255),
 member_type varchar(255)
);
```

### Expected result

```sql
MySQL [(none)]> CREATE DATABASE bikeshare;
Query OK, 0 rows affected (0.14 sec)

MySQL [(none)]> USE bikeshare;
Database changed
MySQL [bikeshare]>
MySQL [bikeshare]> CREATE TABLE trips (
    ->  trip_id bigint NOT NULL PRIMARY KEY auto_increment,
    ->  duration integer not null,
    ->  start_date datetime,
    ->  end_date datetime,
    ->  start_station_number integer,
    ->  start_station varchar(255),
    ->  end_station_number integer,
    ->  end_station varchar(255),
    ->  bike_number varchar(255),
    ->  member_type varchar(255)
    -> );
Query OK, 0 rows affected (0.17 sec)
```

## Load data

This bash step iterates through the `csv` files and imports them into TiDB. It takes approximately *7 minutes*:

```
for file in *.csv
  do echo "== $file ==" && mysql bikeshare -e "LOAD DATA LOCAL INFILE '$file' INTO TABLE trips FIELDS TERMINATED BY ',' ENCLOSED BY '\"' LINES TERMINATED BY '\r\n' IGNORE 1 LINES (duration, start_date, end_date, start_station_number, start_station, end_station_number, end_station, bike_number, member_type);"
done
```

## Check data

Use the MySQL command line client to experiment with the data:

```sql
use bikeshare;
SELECT * FROM trips LIMIT 1;
```

## Delete \*.csv files

You can now safely delete the `csv` and `zip` files which were used to import the sample data:

```
rm 2017*.zip
rm 2017*.csv
```
