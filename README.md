# Distributed File Processing Pipeline with Spark + Kubernetes

This project provides a reproducible, distributed ETL (Extract, Transform, Load) pipeline using **Apache Spark** on a **Kubernetes** cluster. It's designed to process multi-gigabyte datasets, perform complex transformations, and visualize cluster and job performance in near-real-time with **Prometheus** and **Grafana**.

The entire stack can be deployed locally using `kind` or `minikube`, making it an excellent environment for developing, testing, and understanding distributed data processing workflows.


## Problem Statement

The goal is to build a distributed ETL pipeline that can:

  * **Ingest** large, raw CSV/JSON datasets.
  * **Transform** the data through cleaning, enrichment, and aggregations.
  * **Produce** analytics-ready outputs (e.g., Parquet files, summary reports).
  * **Expose** dashboards with near-real-time cluster metrics (e.g., job latency, throughput, resource utilization).

The entire system must be containerized and deployed on a local Kubernetes cluster, with Apache Spark running in a distributed fashion.

-----

## Architecture

The pipeline leverages a modern, cloud-native architecture. The main components are orchestrated by Kubernetes.

```mermaid
graph TD
    subgraph Local Machine
        A[Developer] -- kubectl apply --> B{Kubernetes API Server};
    end

    subgraph Kubernetes Cluster (kind/minikube)
        B -- creates --> C[Spark Driver Pod];
        B -- creates --> D[Spark Executor Pods];
        C -- tasks --> D;
        D -- reads --> E[Input Data @ MinIO/PVC];
        D -- writes --> F[Processed Data @ MinIO/PVC];
        C -- metrics --> G[Prometheus];
        D -- metrics --> G;
        G -- queries --> H[Grafana Dashboard];
    end

    A -- views --> H;
```

  * **Kubernetes (`kind` / `minikube`):** The container orchestrator that manages all the application components, including Spark jobs and the monitoring stack.
  * **Apache Spark:** The distributed processing engine. It runs with a driver pod and multiple executor pods to parallelize transformations.
  * **Storage (MinIO / PVC):** An S3-compatible object store (MinIO) or a Persistent Volume Claim (PVC) is used to store input, intermediate, and final datasets, simulating a cloud storage environment.
  * **Prometheus:** A monitoring system that scrapes metrics exposed by the Spark driver and executors.
  * **Grafana:** A visualization tool that queries Prometheus to display job performance and cluster health on interactive dashboards.
  * **Orchestration:** ETL jobs are run manually via `kubectl` or can be scheduled using a Kubernetes `CronJob`.

-----

## Features

  * **Reproducible Environment:** Fully containerized stack ensures consistency across different machines.
  * **Distributed Processing:** Leverages Spark on Kubernetes for parallel data processing to handle large datasets efficiently.
  * **Comprehensive Monitoring:** Real-time visibility into job performance, resource usage, and data throughput.
  * **Scalability:** Easily scale the number of Spark workers to see the direct impact on performance.
  * **Fault Tolerance:** Demonstrates Spark's ability to recover from worker failures.
  * **Configurable Pipeline:** The ETL logic and job parameters are easily configurable.

-----

## Repository Structure

```
/spark-k8s-pipeline/
├── README.md                 # This file
├── data/                     # Directory for sample or generated datasets
├── experiments/              # Notebooks or scripts for analyzing results
├── infra/
│   ├── kind-cluster.sh       # Script to create a local kind cluster
│   └── deploy.sh             # Script to deploy all infrastructure components
├── k8s/
│   ├── spark-deployment.yaml   # Manifest to run the Spark ETL job
│   ├── prometheus-values.yaml  # Helm values for Prometheus deployment
│   └── grafana-dashboard.json  # Grafana dashboard definition
└── spark_jobs/
    ├── etl_job.py            # Main PySpark ETL application
    └── helpers.py              # Helper functions for the Spark job
```

-----

## Prerequisites

Before you begin, ensure you have the following tools installed on your local machine:

  * [Docker](https://docs.docker.com/get-docker/)
  * [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
  * [Helm](https://helm.sh/docs/intro/install/)
  * [kind](https://www.google.com/search?q=https://kind.sigs.k8s.io/docs/user/quick-start/%23installation) (or [minikube](https://minikube.sigs.k8s.io/docs/start/))

-----

## Getting Started

Follow these steps to set up and run the entire pipeline on your local machine.

### 1\. Clone the Repository

```bash
git clone https://github.com/amar-at-iitm/distributed_file_processing
cd spark-k8s-pipeline
```

### 2\. Create the Local Kubernetes Cluster

This script uses `kind` to bootstrap a Kubernetes cluster with the necessary configuration.

```bash
sh infra/kind-cluster.sh
```

### 3\. Deploy Infrastructure Components

This script uses Helm and `kubectl` to deploy Prometheus, Grafana, and other necessary services to the cluster.

```bash
sh infra/deploy.sh
```

### 4\. Prepare Data

Place your input CSV or JSON files in the `/data` directory. If you don't have large files, you can use a script to generate synthetic data.
*(Note: The deployment must be configured to mount this local directory into the pods or use a tool like MinIO to upload the data.)*

### 5\. Run the Pipeline

Once the cluster is ready and data is available, trigger the ETL job. See the next section for details.

-----

## Running the ETL Job

The Spark ETL job is defined as a Kubernetes manifest. You can start the job by applying this manifest.

```bash
kubectl apply -f k8s/spark-deployment.yaml
```

You can monitor the status of the driver and executor pods:

```bash
# Watch pods in the 'default' namespace (or wherever you run the job)
kubectl get pods -w

# Check logs of the driver pod to see Spark's output
kubectl logs <your-spark-driver-pod-name> -f
```

The job is configured within `k8s/spark-deployment.yaml`. You can modify arguments such as input/output paths, data formats, and other job-specific parameters.

-----

## Scaling the Pipeline

You can evaluate how the pipeline's performance changes with more resources by adjusting the number of Spark executors.

1.  **Edit the deployment file:** Open `k8s/spark-deployment.yaml`.

2.  **Change executor instances:** Modify the `spec.executor.instances` value.

    ```yaml
    # k8s/spark-deployment.yaml
    apiVersion: "sparkoperator.k8s.io/v1beta2"
    kind: SparkApplication
    metadata:
      name: pyspark-etl
      namespace: default
    spec:
      # ... other configs
      executor:
        cores: 1
        instances: 3 # <-- Change this value from 3 to 5, 10, etc.
        memory: "1g"
    ```

3.  **Re-apply the job:**

    ```bash
    kubectl apply -f k8s/spark-deployment.yaml
    ```

Observe the changes in **job latency** and **throughput** on your Grafana dashboard.

-----

## Testing Fault Tolerance

You can simulate a worker failure to verify Spark's resilience.

1.  **Start the ETL job** and wait for the executor pods to be created.
2.  **List the Spark pods:**
    ```bash
    kubectl get pods -l spark-role=executor
    ```
3.  **Delete one of the executor pods:**
    ```bash
    kubectl delete pod <spark-executor-pod-name>
    ```

The Spark driver will detect the lost executor. Kubernetes will automatically restart the pod, and the driver will re-assign the failed tasks to a healthy executor. The job should continue and complete successfully.

-----

## Monitoring and Observability

The pipeline includes a pre-configured Grafana dashboard for visualizing key performance metrics.

### Accessing Grafana

1.  **Forward the Grafana port** to your local machine:
    ```bash
    kubectl port-forward svc/grafana 3000:80 -n monitoring
    ```
2.  Open your web browser and navigate to `http://localhost:3000`.
3.  Log in with the default credentials (`admin` / `prom-operator`). You will be prompted to change the password.
4.  Navigate to the pre-built "Spark Job Monitoring" dashboard.

### Key Dashboard Panels

The dashboard provides charts for:

  * **Job & Stage Duration:** Wall-clock time for the entire job and individual stages.
  * **Task Throughput:** Number of tasks completed per second.
  * **Executor Resource Usage:** CPU and Memory utilization across all executors.
  * **Shuffle I/O:** Data written and read during shuffle operations, which is often a key bottleneck.
  * **Input/Output:** Bytes read and written from the data source and sink.

-----

## License

This project is licensed under the MIT License. See the `LICENSE` file for details.