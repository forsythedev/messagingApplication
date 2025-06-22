# Scalable Real-time Data Ingestion and Processing with Apache Kafka on Google Kubernetes Engine (GKE)

This project outlines how to deploy a highly available and scalable Apache Kafka cluster on Google Kubernetes Engine (GKE), leveraging Google Cloud's persistent storage for data durability. It demonstrates building a robust real-time data pipeline capable of handling high-throughput messaging.

**Assumed File Structure:**
Please ensure you have the following files in your working directory:
* `README.md` (this file)
* `ssd-storageclass.yaml`
* `kafka-values.yaml`
* `producer-deployment.yaml`
* `consumer-deployment.yaml`

---

## 1. Project Overview

This project focuses on deploying Apache Kafka, a distributed streaming platform, across multiple nodes in a GKE cluster. Kafka is essential for building real-time data pipelines, streaming analytics, and microservices communication.

### Why Kafka on Multiple Nodes?

* **Scalability:** Kafka distributes data across multiple broker nodes by dividing topics into partitions. Adding more nodes allows for horizontal scaling of throughput, enabling the cluster to handle increased data volumes.
* **Fault Tolerance & High Availability:** Kafka replicates partitions across different brokers. If a broker (and thus its underlying GKE node) fails, other replicas can seamlessly take over, preventing data loss and ensuring continuous service availability. Production-grade Kafka clusters typically require at least three brokers for robust high availability.
* **Load Distribution:** Producers and consumers interact with various brokers, distributing the network and processing load efficiently across the cluster.

### Why Persistent Storage in Google Cloud?

* **Data Durability:** Kafka persists messages to disk for durability. When deployed on GKE, each Kafka broker pod utilizes a **Persistent Disk (PD)** provided by Google Compute Engine. This ensures that even if a broker restarts or fails, the message data remains intact and available for recovery.
* **Log Retention:** Kafka topics can be configured to retain messages for specified durations or sizes. This persistent message log requires reliable and durable storage, which Google's Persistent Disks readily provide.
* **StatefulSet Requirements:** Kubernetes **StatefulSets** are ideal for stateful applications like Kafka. They guarantee stable, unique network identities and dedicated PersistentVolumes (which map directly to Google Persistent Disks) for each pod, ensuring data consistency and recoverability.

---

## 2. Architecture Overview

The core components of this architecture include:

* **Google Kubernetes Engine (GKE):** The managed Kubernetes service that hosts our cluster.
* **Apache Kafka Brokers:** Deployed as a Kubernetes **StatefulSet**, with each broker pod backed by a dedicated Google Compute Engine **Persistent Disk** for message data.
* **ZooKeeper Ensemble (Optional for older Kafka versions):** If not using KRaft (Kafka Raft), ZooKeeper is deployed as a separate **StatefulSet**, also using Persistent Disks for its metadata. Modern Kafka versions (3.x and newer) integrate this functionality directly, simplifying deployment.
* **Producer Applications:** Kubernetes **Deployments** that send data to Kafka topics. These are typically stateless.
* **Consumer Applications:** Kubernetes **Deployments** that read and process data from Kafka topics. These are also generally stateless.
* **Google Compute Engine Persistent Disks:** The underlying storage for Kafka (and ZooKeeper) pods, dynamically provisioned via **PersistentVolumeClaims (PVCs)**. For high performance, **SSD Persistent Disks** (`pd-ssd`) or **Hyperdisk Throughput** are recommended.
* **Google Cloud Storage (Optional):** Used for archiving processed data, Kafka backups, or storing build artifacts for related applications.
* **Google Cloud Monitoring & Logging:** For observing cluster health, Kafka metrics, and application logs.

+-------------------------------------------------------------------+
|               Google Kubernetes Engine (GKE) Cluster              |
| +-------------------------------------+ +-----------------------+ |
| | Node Pool 1 (Kafka Brokers)         | | Node Pool 2 (Workers) | |
| |                                     | |                       | |
| | +-----------------+                 | | +-------------------+ | |
| | | Kafka Broker 1  |--[PVC/PD]-------->| | Producer App Pod 1| | |
| | | (StatefulSet)   |&lt;--[Network]--+   | | (Deployment)      | | |
| | +-----------------+             |   | +-------------------+ | |
| |                                 |   |                       | |
| | +-----------------+             |   | +-------------------+ | |
| | | Kafka Broker 2  |--[PVC/PD]-------->| | Consumer App Pod 1| | |
| | | (StatefulSet)   |&lt;--[Network]--+   | | (Deployment)      | | |
| | +-----------------+             |   | +-------------------+ | |
| |                                 |   |                       | |
| | +-----------------+             |   | +-------------------+ | |
| | | Kafka Broker 3  |--[PVC/PD]-------->| | Consumer App Pod 2| | |
| | | (StatefulSet)   |&lt;--[Network]--+   | +-------------------+ | |
| | +-----------------+                 | |                       | |
| +-------------------------------------+ +-----------------------+ |
|                                                                   |
| +-----------------------------------+                             |
| | ZooKeeper Ensemble (Optional, for |                             |
| | older Kafka versions / separate)  |                             |
| | +-----------------+               |                             |
| | | ZK Node 1       |--[PVC/PD]----->                             |
| | +-----------------+               |                             |
| | ... (3+ nodes)                    |                             |
| +-----------------------------------+                             |
+-------------------------------------------------------------------+
|                                       |
| (Data Persistence)                    | (Archival/Analytics)
v                                       v
+-----------------------+              +-----------------------+
| Google Compute Engine |              | Google Cloud Storage  |
| Persistent Disks      |              | (Buckets)             |
+-----------------------+              +-----------------------+
|                                       |
| (Managed Database Backend)            | (Data Warehousing)
v                                       v
+-----------------------+              +-----------------------+
| Google Cloud SQL      |              | Google BigQuery       |
+-----------------------+              +-----------------------+


## 3. Prerequisites

Before you begin, ensure you have the following installed and configured:

* **Google Cloud SDK (`gcloud` command-line tool):** [Installation Guide](https://cloud.google.com/sdk/docs/install)
* **`kubectl`:** The Kubernetes command-line tool (comes with GKE or `gcloud components install kubectl`)
* **`helm`:** The Kubernetes package manager (used for deploying Kafka) [Installation Guide](https://helm.sh/docs/intro/install/)

Ensure you are logged into your Google Cloud account and have selected your project:

```bash
gcloud auth login
gcloud config set project YOUR_GCP_PROJECT_ID
```
Replace YOUR_GCP_PROJECT_ID with your actual Google Cloud Project ID.

## 4. Implementation Steps

Follow these steps to deploy your Kafka cluster on GKE. Ensure all .yaml files (`ssd-storageclass.yaml`, `kafka-values.yaml`, `producer-deployment.yaml`, `consumer-deployment.yaml`) are in your current working directory.

### 4.1. Create a GKE Cluster

```bash
gcloud container clusters create kafka-cluster \
    --num-nodes=3 \
    --machine-type=e2-standard-4 \
    --zone=us-central1-a \
    --enable-autoscaling --min-nodes=3 --max-nodes=10 \
    --project YOUR_GCP_PROJECT_ID
    --disk-size=50
```

Wait for the cluster creation to complete. This may take several minutes.

### 4.2. Get Cluster Credentials

```bash
gcloud container clusters get-credentials kafka-cluster --zone=us-central1-a --project YOUR_GCP_PROJECT_ID
```

### 4.3. Define a High-Performance StorageClass (Recommended)

- I saved all of the files from this repo in a storage bucket called _messagingapp_. Copy your files over then start applying the yaml files.

```bash
gsutil cp -r gs://messagingapp/ .
kubectl apply -f ssd-storageclass.yaml
```

### 4.4. Deploy Apache Kafka using Helm

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install my-kafka bitnami/kafka -f kafka-values.yaml --version 23.2.0
```

### 4.5. Verify Your Deployment and Persistent Disks

Check Kafka Pods:

```bash
kubectl get pods -l app.kubernetes.io/name=kafka
```

Check PVCs:

```bash
kubectl get pvc -l app.kubernetes.io/name=kafka
```

Inspect Disks via Console:  
Navigation menu > Compute Engine > Disks.

### 4.6. Deploy Producer and Consumer Applications (Example)

```bash
kubectl apply -f producer-deployment.yaml
kubectl apply -f consumer-deployment.yaml
```

### 4.7. Monitoring and Logging

- **Cloud Logging:** Logs Explorer in the Google Cloud Console  
- **Cloud Monitoring:** Cluster-level metrics  
- For Kafka-specific monitoring, consider Prometheus + Grafana.

## 5. Cleaning Up

```bash
gcloud container clusters delete kafka-cluster --zone=us-central1-a --project YOUR_GCP_PROJECT_ID
```

If reclaimPolicy is `Retain`, manually delete disks from Compute Engine > Disks in the Cloud Console.
