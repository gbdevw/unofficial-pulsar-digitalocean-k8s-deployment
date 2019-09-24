## Deploy Apache Pulsar on your Digital Ocean Kubernetes Cluster (Unofficial)

### Layout

1. About
2. Prerequisites
3. Pricing model
4. Deployment - Quick steps

### About

The scripts aim to deploy a basic Apache Pulsar cluster on a Digital Ocean managed Kubernetes cluster. The two biggest differences with the official scripts are :

1. The use of StatefulSets rather than DaemonSets to provide sticky identities, permanent storage and tune pod scheduling
2. The use of Persistent Volume Claims with a special Storage Class to attach Digital Ocean Block Storage to the pods

The deployment method shown has been tested on a fresh 3 nodes (2vCPU, 4GB RAM) pool. 

Like the official documentation (https://pulsar.apache.org/docs/en/deploy-kubernetes/), this method will deploy :

- A three-node ZooKeeper cluster
- A two-bookie BookKeeper cluster
- A three-broker Pulsar cluster
- A pod from which you can run administrative commands using the pulsar-admin CLI tool

**OPTIONNAL**
- A monitoring stack consisting of Prometheus, Grafana, and the Pulsar dashboard
- NodePort services to expose components to the outside of your Kubernetes cluster (= the whole internet)

### Prerequisites

1. A Digital Ocean Kubernetes cluster
2. kubectl configured to interact with your cluster 

Official documentation for each prerequisite :

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
- 1 Prometheus replica with 10 GiB storage

Price in USD/month = (3 * 20) + (3 * 2 * 0.10) + (2 * (50 + 12) * 0.10) + (10 * 0.10) = 74 USD/month

### Deployment - Quick steps

#### 1. Deploy the ZooKeeper cluster

You can use the following command to deploy your ZooKeeper cluster :

```bash
kubectl apply -f https://raw.githubusercontent.com/guillaume-braibant/unofficial-pulsar-digitalocean-k8s-deployment/master/zookeeper.yaml
```

This command allow you to check the state of your ZooKeeper pods :

```bash
kubectl get pods -l component=zookeeper -o wide
```

NOTE : The first container deployment can take up to several minutes because the container image to pull is big (https://hub.docker.com/r/apachepulsar/pulsar-all/tags)

#### 2. Initialize your ZooKeeper cluster metadata


Once ALL your ZooKeeper pods are in Running State, you can deploy a job that will initialize your ZooKeeper cluster with some metadata.

This command deploys the job that initializes metadata on your ZooKeeper cluster (cluster-metadata.yaml) :

```bash
kubectl apply -f https://raw.githubusercontent.com/guillaume-braibant/unofficial-pulsar-digitalocean-k8s-deployment/master/cluster-metadata.yaml
```

#### 3. Deploy the BookKeeper cluster

This command deploys the BookKeeper cluster :

```bash
kubectl apply -f https://raw.githubusercontent.com/guillaume-braibant/unofficial-pulsar-digitalocean-k8s-deployment/master/bookie.yaml
```

This command allow you to check the state of your BookKeeper pods :

```bash
kubectl get pods -l component=bookkeeper -o wide
```

#### 4. Deploy the brokers

Once your bookies are up, you can deploy the brokers :

```bash
kubectl apply -f https://raw.githubusercontent.com/guillaume-braibant/unofficial-pulsar-digitalocean-k8s-deployment/master/broker.yaml
```

#### 5. Deploy the pulsar-admin pod

This command deploys a pod from which you can run administrative commands using the pulsar-admin CLI tool :

```bash
kubectl apply -f https://raw.githubusercontent.com/guillaume-braibant/unofficial-pulsar-digitalocean-k8s-deployment/master/admin.yaml
```

#### 6. Test your Apache Pulsar cluster

From here, you can run a simple test that uses the pulsar-admin pod to create a producer and consumer :

This command creates the producer :

```bash
kubectl exec pulsar-admin -it -- bin/pulsar-perf produce persistent://public/default/test-topic --rate 1000
```

This command creates the consumer :

```bash
kubectl exec pulsar-admin -it -- bin/pulsar-perf consume persistent://public/default/test-topic --subscriber-name test-subscription
```

#### 7. Deploy the monitoring stack (Optionnal)

The first thing to do is to create a ServiceAccount that allow Prometheus to query your Kubernetes cluster in order to find the different components from which metrics should be scrapped :

```bash
kubectl apply -f https://raw.githubusercontent.com/guillaume-braibant/unofficial-pulsar-digitalocean-k8s-deployment/master/prometheus-rbac.yaml
```

The next step consists in deploying the various components of the monitoring stack :

```bash
kubectl apply -f https://raw.githubusercontent.com/guillaume-braibant/unofficial-pulsar-digitalocean-k8s-deployment/master/monitoring.yaml
```

#### 8. Allow remote access to components inside your Digital Ocean Kubernetes cluster (Optionnal)

**CAUTION : Allowing access from outside your Digital Ocean Kubernetes cluster exposes the components to the whole Internet. Security considerations and component security configuration are beyond the perimeter of this guide.**

Currently, the component of your Apache Pulsar cluster can be accessed only from within your Digital Ocean Kubernetes cluster.

This command deploys a NodePort service that exposes your brokers (30001 & 30002) :

```bash
kubectl apply -f https://raw.githubusercontent.com/guillaume-braibant/unofficial-pulsar-digitalocean-k8s-deployment/master/broker-proxy.yml
```

This command deploys a NodePort service that exposes Prometheus (30003) :

```bash
kubectl apply -f https://raw.githubusercontent.com/guillaume-braibant/unofficial-pulsar-digitalocean-k8s-deployment/master/prometheus-proxy.yml
```

This command deploys a NodePort service that exposes Grafana (30004) :

```bash
kubectl apply -f https://github.com/guillaume-braibant/unofficial-pulsar-digitalocean-k8s-deployment/blob/master/grafana-proxy.yml
```

This command deploys a NodePort service that exposes the Pulsar Dashboard (30005) :

```bash
kubectl apply -f https://raw.githubusercontent.com/guillaume-braibant/unofficial-pulsar-digitalocean-k8s-deployment/master/dashboard-proxy.yml
```

You can access your component by using the Nodeport service port and the public IP of one of the droplets that compose your Digital Ocean cluster (Droplet public IP : port). The ways to provide a less fragile way (not relying on one public IP) to target the components of your Apache Pulsar cluster from outside your Digital Ocean Kubernetes cluster are also beyong the perimeter of this guide. 
