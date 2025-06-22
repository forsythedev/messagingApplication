# Scalable Real-time Data Ingestion and Processing with Apache Kafka on Google Kubernetes Engine (GKE)

This project outlines how to deploy a highly available and scalable Apache Kafka cluster on Google Kubernetes Engine (GKE), leveraging Google Cloud's persistent storage for data durability. It demonstrates building a robust real-time data pipeline capable of handling high-throughput messaging.

[...]

Refer to the included YAML files:
- `ssd-storageclass.yaml`
- `kafka-values.yaml`
- `producer-deployment.yaml`
- `consumer-deployment.yaml`

Use `kubectl apply -f <file>.yaml` to deploy each respective component.
