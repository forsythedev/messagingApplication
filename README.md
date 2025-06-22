# Scalable Real-time Data Ingestion and Processing with Apache Kafka on Google Kubernetes Engine (GKE)

This project outlines how to deploy a highly available and scalable Apache Kafka cluster on Google Kubernetes Engine (GKE), leveraging Google Cloud's persistent storage for data durability. It demonstrates building a robust real-time data pipeline capable of handling high-throughput messaging.

[...]

Refer to the included YAML files:
- `ssd-storageclass.yaml`
- `kafka-values.yaml`
- `producer-deployment.yaml`
- `consumer-deployment.yaml`

Use `kubectl apply -f <file>.yaml` to deploy each respective component.


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
```

Wait for the cluster creation to complete. This may take several minutes.

### 4.2. Get Cluster Credentials

```bash
gcloud container clusters get-credentials kafka-cluster --zone=us-central1-a --project YOUR_GCP_PROJECT_ID
```

### 4.3. Define a High-Performance StorageClass (Recommended)

```bash
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
