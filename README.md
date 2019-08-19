## Deploy Apache Pulsar on your Digital Ocean Kubernetes Cluster (Unofficial)

The scripts on this repository are specialy tailored for a Digital Ocean managed Kubernetes Cluster. The biggest differences with the official scripts is the way storage is managed : With Digital Ocean K8S Clusters, you use Persistent Volume Claims with a special Storage Class in order to get and attach permanent storage to your pods.

The deployment method shown has been tested on a fresh 3-nodes (2vCpu, 4GB) pool. 

Like the official documentation (https://pulsar.apache.org/docs/en/deploy-kubernetes/), this method will deploy :

- A three-node ZooKeeper cluster
- A two-bookie BookKeeper cluster
- A three-broker Pulsar cluster
- A pod from which you can run administrative commands using the pulsar-admin CLI tool
- A monitoring stack consisting of Prometheus, Grafana, and the Pulsar dashboard
- Optionally, NodePort services to expose the different components to the outside your Kubernetes cluster (= the whole internet)
