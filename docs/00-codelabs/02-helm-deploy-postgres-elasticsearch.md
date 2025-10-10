---
layout: default 
title: Deploy Cadence to Google Kubernetes Engine (GKE) with PostgreSQL and Elasticsearch v7 using Helm
permalink: /docs/codelabs/helm-deploy-postgres-elasticsearch
---
## Codelab: Deploy Cadence to Google Kubernetes Engine (GKE) with PostgreSQL and Elasticsearch v7 using Helm

This codelab walks you through deploying Cadence on a Google Cloud GKE cluster using Helm, configuring PostgreSQL as the main database and Elasticsearch v7 for advanced visibility. It is designed for first-time Helm and Kubernetes users.

- Prerequisites: Google Cloud project with billing enabled

### What you’ll build
- A GKE cluster
- A Cadence deployment (frontend, history, matching, worker)
- PostgreSQL as the main persistence store
- Elasticsearch v7 for advanced visibility
- Cadence Web UI



---

### Step 0: Setup your local tools

Install or verify the following tools:

```bash
# Install gcloud
brew install --cask google-cloud-sdk

# Install kubectl
brew install kubectl

# Install Helm
brew install helm
```

Authenticate GCloud and set your project/region/zone:

```bash
gcloud init
# Or set directly
gcloud auth login
gcloud config set project <YOUR_GCP_PROJECT_ID>
gcloud config set compute/region us-central1
gcloud config set compute/zone us-central1-a
```

Get the Cadence Helm charts repository locally:

```bash
# Clone the repo and enter it (official repo)
git clone https://github.com/cadence-workflow/cadence-charts.git
cd cadence-charts
```

Optional: use the remote Helm repo instead of cloning (from the official README)
([source](https://github.com/cadence-workflow/cadence-charts.git)):

```bash
helm repo add cadence https://cadence-workflow.github.io/cadence-charts
helm repo update
# Then install with: helm install my-cadence cadence/cadence
# (This codelab uses the local chart path to keep files together.)
```

---

### Step 1: Create a GKE cluster and connect kubectl

```bash
# Create a small test cluster (adjust as needed)
gcloud container clusters create cadence-gke \
  --num-nodes=3 \
  --machine-type=e2-standard-2 \
  --enable-ip-alias

# Get cluster credentials for kubectl
gcloud container clusters get-credentials cadence-gke

# Verify connection
kubectl get nodes
```

Create a namespace for Cadence:

```bash
kubectl create namespace cadencetest
```

---

### Step 2: Prepare a Helm values file for PostgreSQL + Elasticsearch v7

In this repository, create `examples/gke-postgres-es7-values.yaml` with the following content:

```yaml
# Namespace note: install into namespace: cadencetest

# Force Cadence to use PostgreSQL for main DB
config:
  persistence:
    database:
      driver: "postgres"
      sql:
        hosts: "cadence-release-postgresql.cadencetest.svc.cluster.local"
        port: 5432
        dbname: "cadence"
        visibilityDbname: "cadence_visibility"
        user: "cadence"
        password: "changeme-strong"
        tls:
          enabled: false
          sslMode: ""
    elasticsearch:
      enabled: true
      version: "v7"
      user: ""
      password: ""
      protocol: "http"
      hosts: "elasticsearch-master.elastic-system.svc.cluster.local"
      port: 9200
      visibilityIndex: "cadence-visibility"
      tls:
        enabled: false

# Enable Cadence schema jobs
schema:
  serverJob:
    enabled: true
  elasticSearchJob:
    enabled: true

# Ensure Cadence uses Elasticsearch for advanced visibility
# See values.yaml dynamicConfig keys in the chart
dynamicConfig:
  values:
    system.writeVisibilityStoreName:
      - value: "es-visibility"
    system.readVisibilityStoreName:
      - value: "es-visibility"

# Deploy Postgres within the same release (Bitnami subchart)
postgresql:
  enabled: true
  auth:
    username: cadence
    password: "changeme-strong"
    database: cadence
  primary:
    persistence:
      enabled: true
      size: 8Gi

# Do NOT deploy Cassandra or MySQL
cassandra:
  enabled: false
mysql:
  enabled: false

# Optional: Cadence Web UI config (ClusterIP by default)
web:
  replicas: 1
  service:
    type: ClusterIP
```

Notes:
- This values file: switches main DB to PostgreSQL; enables Elasticsearch v7; turns on schema jobs.
- It deploys a Bitnami PostgreSQL as part of this Helm release for convenience.
- You must point `config.persistence.elasticsearch.hosts` to your Elasticsearch v7 cluster DNS or endpoint.

If you don’t have Elasticsearch yet, consider these options:
- Managed: [Elastic Cloud on GCP](https://www.elastic.co/cloud/)
- Self-managed Helm: [Bitnami Elasticsearch](https://artifacthub.io/packages/helm/bitnami/elasticsearch)

---

### Step 3: Add Helm repos (if needed) and install Cadence

From the repository root:

```bash
# Ensure you are in this repo root
cd cadence-charts

# (If using remote repos) Add and update Helm repos
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Install Cadence with our values file
helm upgrade --install cadence-release ./charts/cadence \
  --namespace cadencetest \
  -f examples/gke-postgres-es7-values.yaml \
  --wait
```

Verify resources:

```bash
kubectl get pods -n cadencetest
kubectl get svc -n cadencetest
```

Wait until all pods are `Running` or `Completed` (schema jobs should complete then disappear).

---

### Step 4: Verify schema jobs and Cadence health

Check schema jobs ran successfully:

```bash
# If jobs are still present, check their status
kubectl get jobs -n cadencetest
kubectl logs job/cadence-release-cadence-schema-server -n cadencetest --tail=200 || true
kubectl logs job/cadence-release-cadence-schema-elasticsearch -n cadencetest --tail=200 || true
```

Port-forward frontend and web to your local machine:

```bash
# Frontend gRPC (7833) and TChannel (7933)
kubectl port-forward -n cadencetest svc/cadence-release-cadence-frontend 7833:7833 7933:7933
```

In a new terminal, port-forward the Web UI:

```bash
kubectl port-forward -n cadencetest svc/cadence-release-cadence-web 8088:8088
```

Open the Cadence Web UI at `http://localhost:8088`.

---

### Step 5: Create a sample domain and run a quick check

Install the Cadence CLI (tctl) locally or exec into the frontend pod. Docs: [Cadence CLI](https://cadenceworkflow.io/docs/cli/)

Using kubectl exec:

```bash
# Find a frontend pod
POD=$(kubectl get pods -n cadencetest -l app.kubernetes.io/component=frontend -o jsonpath='{.items[0].metadata.name}')
```
```bash
# Register a domain
kubectl exec -n cadencetest -it "$POD" -- \
  tctl --address cadence-release-cadence-frontend:7833 \
  --do sample-domain domain register -rd 1
```
```bash
# List domains
kubectl exec -n cadencetest -it "$POD" -- \
  tctl --address cadence-release-cadence-frontend:7833 domain list
```

If you enabled Elasticsearch v7 and pointed `hosts` to an active cluster, Cadence will use ES for advanced visibility. Confirm index:

```bash
# Example if you can curl ES from your laptop
echo "Check your ES index:" && \
  curl -s http://<ES_HOST>:9200/cadence-visibility
```

---

### Step 6: Uninstall and cleanup (optional)

```bash
# Remove the Helm release
helm uninstall cadence-release -n cadencetest

# Optionally delete the namespace
kubectl delete namespace cadencetest

# Optionally delete the GKE cluster (this is destructive!)
gcloud container clusters delete cadence-gke --quiet
```

---

### Troubleshooting
- Postgres connection fails: confirm `postgresql.enabled: true` and that `username/password/database` match the `config.persistence.database.sql` section.
- ES visibility not working: verify `config.persistence.elasticsearch.enabled: true`, version `v7`, and reachable `hosts`/`port`.
- Pods stuck pending: check node resources and quotas; use `kubectl describe pod`.
- Jobs failing: inspect logs with `kubectl logs job/<job-name> -n cadencetest`.

### References 
- Cadence Helm charts repo: [cadence-workflow/cadence-charts](https://github.com/cadence-workflow/cadence-charts.git)
- Cadence Helm chart README: [Cadence Charts README](https://github.com/cadence-workflow/cadence-charts/tree/main/charts/cadence)
- Helm docs: [Install Helm](https://helm.sh/docs/intro/install/)
- GKE docs: [Creating a cluster](https://cloud.google.com/kubernetes-engine/docs/how-to/creating-a-zonal-cluster)
- Kubernetes docs: [kubectl overview](https://kubernetes.io/docs/reference/kubectl/)
- Cadence server docs: [Cadence Docs](https://cadenceworkflow.io/docs/)
