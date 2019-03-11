---
title: Deploy KOST Stack on GKE
summary: Tutorial for deploying TiDB on Google Cloud using Kubernetes.
category: operations
---

# Deploy KOST Stack on GKE

## Introduction

This tutorial is designed to be [run in Google Cloud Shell](https://console.cloud.google.com/cloudshell/open?git_repo=https://github.com/pingcap/tidb-operator&tutorial=docs/google-kubernetes-tutorial.md). It takes you through these steps:

- Launch a new 3-node Kubernetes cluster
- Install the Helm package manager for Kubernetes
- Deploy the TiDB Operator
- Deploy your first TiDB cluster
- Connect to the TiDB cluster
- Create shortcuts

## Select a project

This tutorial launches a 3-node Kubernetes cluster of `n1-standard-1` machines. Pricing information can be [found here](https://cloud.google.com/compute/pricing).

Please select a project before proceeding:

<walkthrough-project-billing-setup key="project-id">
</walkthrough-project-billing-setup>

## Enable API access

This tutorial requires the use of the Compute and Container APIs. Please enable them before proceeding:

<walkthrough-enable-apis apis="container.googleapis.com,compute.googleapis.com">
</walkthrough-enable-apis>

## Configure gcloud defaults

This step defaults gcloud to your preferred project and [zone](https://cloud.google.com/compute/docs/regions-zones/), which will simplify the commands used for the rest of this tutorial:

	gcloud config set project {{project-id}} &&
	gcloud config set compute/zone us-west1-a

## Launch a 3-node Kubernetes cluster

Before you begin, make sure you are running the latest version of tools in your Google Cloud Shell:

	sudo gcloud components update

It's now time to launch a 3-node kubernetes cluster! The following command launches a 3-node cluster of `n1-standard-1` machines.

It will take a few minutes to complete:

	gcloud container clusters create tidb

Once the cluster has launched, set it to be the default:

	gcloud config set container/cluster tidb

The last step is to verify that `kubectl` can connect to the cluster, and all three machines are running:

	kubectl get nodes

If you see `Ready` for all nodes, congratulations! You've setup your first Kubernetes cluster.

## Install Helm

Helm is the package manager for Kubernetes, and is what allows us to install all of the distributed components of TiDB in a single step. Helm requires both a server-side and a client-side component to be installed.

Download the `helm` installer:

	curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get > get_helm.sh

Install `helm`:

	chmod 700 get_helm.sh && ./get_helm.sh

Copy `helm` to your `$HOME` directory so that it will persist after the Cloud Shell reaches its idle timeout:

	mkdir -p ~/bin &&
	cp /usr/local/bin/helm ~/bin &&
	echo 'PATH="$PATH:$HOME/bin"' >> ~/.bashrc

Helm will also need a couple of permissions to work properly. We can download them from the `tidb-operator` project:

	git clone https://github.com/pingcap/tidb-operator.git && cd tidb-operator &&
	kubectl create serviceaccount tiller --namespace kube-system &&
	kubectl apply -f ./manifests/tiller-rbac.yaml &&
	helm init --service-account tiller --upgrade

It takes a minute for helm to initialize `tiller`, its server component:

	watch "kubectl get pods --namespace kube-system | grep tiller"

When you see `Running`, it's time to hit `Control + C` and proceed to the next step!

## Deploy TiDB Operator

The first TiDB component we are going to install is the TiDB Operator, using a Helm Chart. TiDB Operator is the management system that works with Kubernetes to bootstrap your TiDB cluster and keep it running. This step assumes you are in the `tidb-operator` working directory:

	KUBE_VERSION=$(kubectl version --short | awk '/Server/{print $NF}' | awk -F '-' '{print $1}') &&
	kubectl apply -f ./manifests/crd.yaml &&
	kubectl apply -f ./manifests/gke-storage.yml &&
	helm install ./charts/tidb-operator -n tidb-admin --namespace=tidb-admin --set scheduler.kubeSchedulerImage=gcr.io/google-containers/hyperkube:${KUBE_VERSION}

We can watch the operator come up with:

	watch kubectl get pods --namespace tidb-admin -o wide

When you see `Running`, `Control + C` and proceed to launch a TiDB cluster!

## Deploy your first TiDB cluster

Now with a single command, we can bring-up a full TiDB cluster. We are overwriting some of the defaults here to use TiDB 2.1, which is currently a release candidate:

	helm install ./charts/tidb-cluster -n tidb --namespace=tidb --set pd.storageClassName=pd-ssd,tikv.storageClassName=pd-ssd,pd.image=pingcap/pd:latest,tidb.image=pingcap/tidb:latest,tikv.image=pingcap/tikv:latest

It will take a few minutes to launch. You can monitor the progress with:

	watch kubectl get pods --namespace tidb -o wide

The TiDB cluster includes 2 TiDB pods, 3 TiKV pods, and 3 PD pods. When you see all pods `Running`, it's time to `Control + C` and proceed forward!

## Connect to the TiDB cluster

There can be a small delay between the pod being up and running, and the service being available. You can watch list services available with:

	watch "kubectl get svc -n tidb"

When you see `demo-tidb` appear, you can `Control + C`. The service is ready to connect to!

You can connect to the clustered service within the Kubernetes cluster:

	kubectl run -n tidb mysql-client --rm -i --tty --image mysql -- mysql -P 4000 -u root -h $(kubectl get svc demo-tidb -n tidb --output json | jq -r '.spec.clusterIP')

Congratulations, you are now up and running with a distributed TiDB database compatible with MySQL!

If you like, you can check your connection to TiDB:

	select tidb_version();

In addition to connecting to TiDB within the Kubernetes cluster, you can also establish a tunnel between the TiDB service and your Cloud Shell. This is recommended only for debugging purposes, because the tunnel will not automatically be transferred if your Cloud Shell restarts. To establish a tunnel:

	kubectl -n tidb port-forward demo-tidb-0 4000:4000 &>/dev/null &

From your Cloud Shell:

	sudo apt-get install -y mysql-client &&
	mysql -h 127.0.0.1 -u root -P 4000

## Create shortcuts

To ensure the defaults are saved in your cloud shell, create the file `~/bin/establish-port-forward` with the following contents:

```
cat << EOF > ~/bin/establish-port-forward
#!/bin/sh
gcloud config set project {{project-id}}
gcloud config set compute/zone us-west1-a
gcloud config set container/cluster tidb

# Setup port forward and reinstall mysql client if required.
kubectl -n tidb port-forward demo-tidb-0 4000:4000 &>/dev/null &
which mysql || apt-get install -y mysql-client
EOF

chmod +x ~/bin/establish-port-forward
```

Write a `~/.my.cnf` config file:

```
cat << EOF > ~/.my.cnf
[client]
user=root
port=4000
host=127.0.0.1
EOF
```
	
Now when you resume work, you just need to type `establish-port-forward` and then `mysql` without any arguments.
