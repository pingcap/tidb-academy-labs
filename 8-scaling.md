---
title: Scaling TiDB
summary: Tutorial for deploying TiDB on Google Cloud using Kubernetes.
category: operations
---

# Scaling	

## Introduction

Using your previously deployed Kubernetes Cluster, we will complete the following steps:

- Scale the size of the Kubernetes cluster
- Scale the number of TiDB servers
- Scaling the number of TiKV servers
- Scale back down to previous size

## Recheck configuration

Establish a tunnel to TiDB:

	establish-port-forward
	
If you see `-bash: establish-port-forward: command not found`, reload the setup tutorial and make sure your environment is configured correctly.

## Scale up Kubernetes

The following command scales the size of your cluster up to 4 `n1-standard` instances. Unless your cluster already has free capacity, it is typical to scale up Kubernetes before scaling up the number of TiDB or TiKV servers:

	gcloud container clusters resize tidb --size 4
	
This operation will take a couple of minutes.

## Scale up TiDB Servers

Make sure you are in the directory which contains tidb-operator. If you need to, you can check it out first:

```
git clone https://github.com/pingcap/tidb-operator.git
cd tidb-operator
helm upgrade tidb charts/tidb-cluster --set pd.storageClassName=pd-ssd,tikv.storageClassName=pd-ssd,pd.image=pingcap/pd:v2.1.5,tidb.image=pingcap/tidb:v2.1.5,tikv.image=pingcap/tikv:v2.1.5,tidb.replicas=4
```

You can monitor the replicas starting with:

```
kubectl get po -n tidb
```

## Scale up TiKV Servers

Make sure you are in the directory which contains tidb-operator:

```
helm upgrade tidb charts/tidb-cluster --set pd.storageClassName=pd-ssd,tikv.storageClassName=pd-ssd,pd.image=pingcap/pd:v2.1.5,tidb.image=pingcap/tidb:v2.1.5,tikv.image=pingcap/tikv:v2.1.5,tidb.replicas=4,tikv.replicas=5
kubectl get po -n tidb
```

## Scale down to previous size

To reduce operating costs, scale your Kubernetes cluster back down to 3 instances, and TiDB back down to 2:

```
helm upgrade tidb charts/tidb-cluster --set pd.storageClassName=pd-ssd,tikv.storageClassName=pd-ssd,pd.image=pingcap/pd:v2.1.5,tidb.image=pingcap/tidb:v2.1.5,tikv.image=pingcap/tikv:v2.1.5,tidb.replicas=2,tikv.replicas=3
gcloud container clusters resize tidb --size 3
```
