---
title: Observing TiDB Cluster Metrics
summary: Tutorial for deploying TiDB on Google Cloud using Kubernetes.
category: operations
---

# Observing TiDB Cluster Metrics

## Introduction

Using your previously deployed Kubernetes Cluster, we will complete the following steps:

- Setup a load balancer to route Grafana to the public internet
- Connect to Grafana
- Observe metrics being fed by TiDB servers

## Expose monitor using a load balancer

The Grafana installation is locally available inside the Kubernetes cluster.  Previously when we needed to connect to a TiDB server pod, we've setup port forwarding to our cloud shell.

In this instance though, we want to be able to connect with our web-browser, so what we are going to do is to set up a new service which is an external load balancer, and plug the "monitor" pod behind it. Because this lesson revolves around a Graphical interface, we are going to do this using the Google Cloud Shell.

Go to [console.cloud.google.com](https://console.cloud.google.com) and navigate to _Kubernetes Engine_. Click on Workloads, and then find the _demo-monitor_. This is the Grafana deployment that comes with TiDB:

![alt text](https://github.com/pingcap/tidb-academy-labs/raw/master/mysql_dbas/workloads.png "Workloads")

After clicking on _demo-monitor_ select _Expose_ from the actions menu:

![alt text](https://github.com/pingcap/tidb-academy-labs/raw/master/mysql_dbas/actions.png "Actions > Expose")

You want to expose port 3000 to the Service type _Load balancer_:

![alt text](https://github.com/pingcap/tidb-academy-labs/raw/master/mysql_dbas/expose.png "Actions > Expose")

You have now created a service to expose Grafana externally. It will take a minute for the public IP address of the load balancer to appear:

![alt text](https://github.com/pingcap/tidb-academy-labs/raw/master/mysql_dbas/waiting.png "Actions > Expose")

Hit refresh if you need to. Once the note about "Creating Service Endpoints" disappears, scroll down to find the load balancer's public IP address:

![alt text](https://github.com/pingcap/tidb-academy-labs/raw/master/mysql_dbas/final.png "Actions > Expose")

Note the IP address. In my case it was *35.197.118.183*.

## Browse to Grafana

Using the IP address provided in the previous step, connect to Grafana.

In my case, I opened http://35.197.118.183:3000/. 

The default username is "admin" with password "admin".

![alt text](https://github.com/pingcap/tidb-academy-labs/raw/master/mysql_dbas/grafana.png "Actions > Expose")

## Select the TiDB Dashboard

When you have logged in, select the *TiDB-Cluster-TiDB* dashboard:
![alt text](https://github.com/pingcap/tidb-academy-labs/raw/master/mysql_dbas/select-dashboard.png "Actions > Expose")
![alt text](https://github.com/pingcap/tidb-academy-labs/raw/master/mysql_dbas/tidb-dashboard.png "Actions > Expose")

From here you can see that the TiDB servers have already been feeding metrics to Prometheus. We can see the aggregated statistics between demo-tidb-0 and demo-tidb-1:

![alt text](https://github.com/pingcap/tidb-academy-labs/raw/master/mysql_dbas/tidb-overview.png "Actions > Expose")

## Suggested metrics to look at

There are [suggested metrics to look at](https://pingcap.com/docs/op-guide/dashboard-overview-info/) in the TiDB manual. For example, as a general load number you can look at _QPS By instance_ or _connection count_.

## Additional features of Grafana

Grafana can also be used to trigger alerts, which a few have been configured for you already.

![alt text](https://github.com/pingcap/tidb-academy-labs/raw/master/mysql_dbas/alerts.png "Actions > Expose")

Alerts can be configured to send to your favourite communication channel. There is email of course, but in a modern scenario, integrations like PagerDuty and Slack are typically more useful.
