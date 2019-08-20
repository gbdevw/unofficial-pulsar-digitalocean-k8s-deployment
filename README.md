## Deploy Apache Pulsar on your Digital Ocean Kubernetes Cluster (Unofficial)

### About

The scripts aim to deploy a basic Apache Pulsar Cluster on a Digital Ocean managed Kubernetes cluster. The biggest differences with the official scripts is the way storage is managed : With Digital Ocean K8S clusters, you use Persistent Volume Claims with a special Storage Class in order to get and attach permanent storage to your pods.

The deployment method shown has been tested on a fresh 3 nodes (2vCPU, 4GB RAM) pool. 

Like the official documentation (https://pulsar.apache.org/docs/en/deploy-kubernetes/), this method will deploy :

MANDATORY :
- A three-node ZooKeeper cluster
- A two-bookie BookKeeper cluster
- A three-broker Pulsar cluster

OPTIONNAL :
- A pod from which you can run administrative commands using the pulsar-admin CLI tool
- A monitoring stack consisting of Prometheus, Grafana, and the Pulsar dashboard
- NodePort services to expose components to the outside of your Kubernetes cluster (= the whole internet)

### Prerequisites

1. A Digital Ocean Kubernetes cluster
2. kubectl configured to interact with your cluster 

Official documentation for each step :

1. https://www.digitalocean.com/docs/kubernetes/how-to/create-clusters/
2. https://www.digitalocean.com/docs/kubernetes/how-to/connect-to-cluster/

### Pricing model

THIS MODEL DOES NOT TAKE INTO ACCOUNT BANDWIDTH USAGE AND BILLING !

**Price per month in USD = (Nprice * N) + (Z * Zstorage * 0.10) + (B * (Bjournal + Bledger) * 0.10) + (Pstorage * 0.10)**

- N : number of nodes
- Nprice : price in USD/month of a node
- Z : Number of ZooKeeper replicas
- Zstorage : Size of the storage (GiB) requested by a ZooKeeper replica
- B : Number of BookKeeper replicas
- Bjournal : Size of the storage (GiB) requested by a BookKeeper replica for the journal 
- Bledger : Size of the storage (GiB) requested by a BookKeeper replica for the ledger 
- Pstorage : Size of the storage (GiB) requested by the Prometheus replica
- Block storage is currently billed 0.10 USD per month per GiB

**Pricing for the example**

- 3 Nodes billed 20 USD/month
- 3 ZooKeeper replicas with 2GiB storage each
- 2 BookKeeper replicas with 50GiB storage for ledger + 12 GiB storage for journal
- 1 Prometheus replicas with 10 GiB storage

Price in USD/month = (3 * 20) + (3 * 2 * 0.10) + (2 * (50 + 12) * 0.10) + (10 * 0.10) = 74 USD/month

### Deployment (summary)

TODO

### Deployment (explained)

All component are deployed in the default namespace. All components are labelled with app=pulsar and various specific labels and values (component, storage, ...).

#### 1. Deploy the ZooKeeper cluster

The first thing to do is to deploy the ZooKeeper cluster that will help coordinating the different components of your Apache Pulsar cluster. The zookeeper.yaml file deploys a headless service and a 3 ZooKeeper replicas StatefulSet. 

Using a StatefulSet provides sticky identities to your ZooKeeper pods, allow them to claim a block storage and ensure that each ZooKeeper pod can retrieve its storage when the pod is rescheduled, restarted, etc... The StatefulSet use a pod anti-affinity rule in order to schedule the ZooKeeper pods on different nodes if possible.

You can use the following command to deploy your ZooKeeper cluster :

```bash
kubectl apply -f https://raw.githubusercontent.com/guillaume-braibant/unofficial-pulsar-digitalocean-k8s-deployment/master/zookeeper.yaml
```

This command allow you to check the state of your ZooKeeper pods :

```bash
kubectl get pods -l component=zookeeper -o wide
```

Once ALL your ZooKeeper pods are in Running State, you can deploy a job that will initialize your ZooKeeper cluster with some metadata.

NOTE : The first container deployment can take up to several minutes because the container image to pull is big (https://hub.docker.com/r/apachepulsar/pulsar-all/tags)

#### 2. Initialize your ZooKeeper cluster metadata

This command deploys the job (cluster-metadata.yaml) :

```bash
kubectl apply -f https://raw.githubusercontent.com/guillaume-braibant/unofficial-pulsar-digitalocean-k8s-deployment/master/cluster-metadata.yaml
```


