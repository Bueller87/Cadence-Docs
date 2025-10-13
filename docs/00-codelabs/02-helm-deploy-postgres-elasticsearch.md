---
layout: default 
title: Deploy Cadence with Helm on GKE
permalink: /docs/codelabs/helm-deploy-postgres-elasticsearch
---
## Codelab: Deploy Cadence with Helm on GKE

Ready to deploy Cadence? This codelab walks you through a complete, production-ready deployment using official Helm charts—no custom scripting required. Whether you're new to Kubernetes or an experienced platform engineer, you'll have Cadence running on GKE with PostgreSQL and Elasticsearch in under 30 minutes.

We've done the heavy lifting: the Helm charts are tested, maintained, and ready to use. Just follow the steps, and you'll have a fully functional Cadence cluster with advanced visibility features.

**Prerequisites:**
- Google Cloud project with billing enabled
- An existing GKE cluster with kubectl access
  - If you need to create a cluster, see [Creating a GKE cluster](https://cloud.google.com/kubernetes-engine/docs/how-to/creating-a-zonal-cluster)
- Basic familiarity with terminal commands

**Designed for:**
- First-time Helm and Kubernetes users who want clear explanations
- Experienced users who need a quick reference

### What you'll build
- A complete Cadence deployment (frontend, history, matching, worker)
- PostgreSQL as the main persistence store
- Elasticsearch v7 for advanced visibility
- Cadence Web UI for workflow monitoring

---

### Step 0: Setup your local tools

This step ensures you have the necessary tools installed and configured. The examples below use macOS commands, but the official documentation links provide instructions for all platforms.

#### Prerequisites

You'll need these tools installed locally:

**Google Cloud SDK (gcloud)** - Manages GCP resources and authentication
- Install instructions: https://cloud.google.com/sdk/docs/install

**kubectl** - Kubernetes command-line tool for cluster management
- Install instructions: https://kubernetes.io/docs/tasks/tools/

**Helm** - Kubernetes package manager
- Install instructions: https://helm.sh/docs/intro/install/

**Quick install (macOS with Homebrew):**

```bash
brew install --cask google-cloud-sdk
```

```bash
brew install kubectl
```

```bash
brew install helm
```

#### Authenticate and configure GCloud

Initialize gcloud (interactive setup):

```bash
gcloud init
```

Or configure manually:

Login to your Google account:

```bash
gcloud auth login
```

Set your GCP project (replace `<YOUR_GCP_PROJECT_ID>` with your actual project ID):

```bash
gcloud config set project <YOUR_GCP_PROJECT_ID>
```

Set your preferred region:

```bash
gcloud config set compute/region us-central1
```

Set your preferred zone:

```bash
gcloud config set compute/zone us-central1-a
```

#### Get Cadence Helm Charts

This codelab uses the local charts approach. Clone the repository:

```bash
git clone https://github.com/cadence-workflow/cadence-charts.git
```

```bash
cd cadence-charts
```

**Alternative approach:** Use the remote Helm repository (not used in this codelab):

```bash
helm repo add cadence https://cadence-workflow.github.io/cadence-charts
```

```bash
helm repo update
```

Note: If using the remote repo, replace `./charts/cadence` with `cadence/cadence` in install commands.

---

### Step 1: Connect to your GKE cluster and create a namespace

**Prerequisites:** You need an existing GKE cluster. In many organizations, platform or infra teams manage cluster creation. If you need to create a cluster yourself, see the official documentation: [Creating a GKE cluster](https://cloud.google.com/kubernetes-engine/docs/how-to/creating-a-zonal-cluster)

For this guide, we'll assume your cluster is named `cadence-gke`. Replace this with your actual cluster name in the commands below.

#### Connect kubectl to your cluster

Get credentials for your existing GKE cluster (replace `cadence-gke` with your cluster name):

```bash
gcloud container clusters get-credentials cadence-gke
```

Verify the connection by listing nodes:

```bash
kubectl get nodes
```

You should see a list of nodes in your cluster with a `Ready` status.

#### Create a namespace for Cadence

Namespaces provide logical isolation within the cluster:

```bash
kubectl create namespace cadence-codelab
```

Verify the namespace was created:

```bash
kubectl get namespace cadence-codelab
```

---

### Step 2: Prepare a Helm values file for PostgreSQL + Elasticsearch v7

This step creates a custom Helm values file to configure Cadence with PostgreSQL as the main database and Elasticsearch v7 for advanced visibility features.

#### Prerequisites: Elasticsearch cluster endpoint

**Important:** You need an existing Elasticsearch v7 cluster. In most organizations, a managed Elasticsearch cluster is provided by your platform or infra team.

Get your Elasticsearch endpoint information:
- **Hostname/URL:** The DNS name or IP address of your ES cluster
- **Port:** Usually 9200 (HTTP) or 9243 (HTTPS for managed services)
- **Credentials:** Username and password if authentication is enabled
- **TLS settings:** Whether HTTPS is required

**Common Elasticsearch scenarios:**

**1. ES running in the same Kubernetes cluster (different namespace):**
```yaml
hosts: "your-es-service.your-es-namespace.svc.cluster.local"
port: 9200
protocol: "http"
```

**2. Managed Elasticsearch service (like Elastic Cloud):**
```yaml
hosts: "your-cluster-id.es.us-central1.gcp.cloud.es.io"
port: 9243
protocol: "https"
user: "your-username"
password: "your-password"
tls:
  enabled: true
```

**3. External Elasticsearch host:**
```yaml
hosts: "es.yourcompany.com"
port: 9200
protocol: "http"
```

If you don't have Elasticsearch yet, consider these options:
- Managed: [Elastic Cloud on GCP](https://www.elastic.co/cloud/)
- Self-managed Helm: [Bitnami Elasticsearch](https://artifacthub.io/packages/helm/bitnami/elasticsearch)

#### Create the values file

In the `cadence-charts` directory, create `examples/gke-postgres-es7-values.yaml` with the following content.

**Before using this file, you MUST update the Elasticsearch `hosts` value** (line 24) with your actual ES endpoint:

```yaml
# Namespace note: install into namespace: cadence-codelab

# Force Cadence to use PostgreSQL for main DB
config:
  persistence:
    database:
      driver: "postgres"
      sql:
        hosts: "cadence-release-postgresql.cadence-codelab.svc.cluster.local"
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
      hosts: "YOUR-ES-HOSTNAME-HERE"  # REPLACE with your actual ES endpoint
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

# Cadence Web UI config (ClusterIP by default)
web:
  replicas: 1
  service:
    type: ClusterIP
```

#### What this configuration does

**Database (PostgreSQL):**
- Deploys a Bitnami PostgreSQL instance as part of this Helm release
- Creates two databases: `cadence` (main) and `cadence_visibility`
- PostgreSQL hostname is auto-generated based on release name and namespace

**Elasticsearch (Advanced Visibility):**
- Enables Elasticsearch v7 for advanced workflow search and filtering
- Configures Cadence to write and read visibility data from ES
- You must provide your own ES cluster endpoint

**Schema Jobs:**
- Automatically runs database schema setup for PostgreSQL
- Automatically runs index setup for Elasticsearch
- Jobs run once during installation and complete

**Cadence Web UI:**
- Deploys the web interface as a ClusterIP service
- Accessible via port-forward (shown in later steps)

---

### Step 3: Install Cadence with Helm

This step installs Cadence and its dependencies (PostgreSQL) into your cluster using the values file you created.

#### Add Helm repositories

First, ensure you're in the `cadence-charts` directory:

```bash
cd cadence-charts
```

Add the Bitnami repository (needed for the PostgreSQL subchart):

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```

Update Helm repositories to get the latest chart versions:

```bash
helm repo update
```

#### Install Cadence

Install Cadence using the values file from Step 2. This command will:
- Create a Helm release named `cadence-release`
- Deploy to the `cadence-codelab` namespace
- Install PostgreSQL, Cadence services (frontend, history, matching, worker), and Cadence Web
- Run schema initialization jobs
- Wait until all resources are ready (typically 2-5 minutes)

```bash
helm upgrade --install cadence-release ./charts/cadence \
  --namespace cadence-codelab \
  -f examples/gke-postgres-es7-values.yaml \
  --wait
```

The `--wait` flag keeps the command running until all pods are ready. You'll see status messages as resources are created.

#### Verify installation

Check that all pods are running:

```bash
kubectl get pods -n cadence-codelab
```

You should see pods for:
- `cadence-release-cadence-frontend`
- `cadence-release-cadence-history`
- `cadence-release-cadence-matching`
- `cadence-release-cadence-worker`
- `cadence-release-cadence-web`
- `cadence-release-postgresql`
- Schema jobs (may show as `Completed` or already gone)

All pods should show `Running` status (or `Completed` for jobs).

Check services:

```bash
kubectl get svc -n cadence-codelab
```

You should see services for frontend, web, and PostgreSQL.

---

### Step 4: Verify schema jobs and Cadence health

This step confirms that database schemas were initialized correctly and sets up local access to Cadence services.

#### Check schema jobs

Schema jobs run once during installation to set up database tables and Elasticsearch indices. Check if the jobs completed successfully:

```bash
kubectl get jobs -n cadence-codelab
```

You should see jobs with `COMPLETIONS` showing `1/1`. If jobs are still present, check their logs:

View PostgreSQL schema job logs:

```bash
kubectl logs job/cadence-release-cadence-schema-server -n cadence-codelab --tail=200
```

View Elasticsearch schema job logs:

```bash
kubectl logs job/cadence-release-cadence-schema-elasticsearch -n cadence-codelab --tail=200
```

Look for messages indicating successful schema creation. If you see errors, they typically relate to connectivity issues with PostgreSQL or Elasticsearch.

#### Access Cadence services locally

Port-forwarding creates a tunnel from your local machine to services running in the cluster. This allows you to interact with Cadence as if it were running locally.

**Port-forward the Cadence frontend** (for CLI access):

```bash
kubectl port-forward -n cadence-codelab svc/cadence-release-cadence-frontend 7833:7833
```

This command runs in the foreground. Keep it running and open a new terminal for the next command.

**In a new terminal, port-forward the Cadence Web UI:**

```bash
kubectl port-forward -n cadence-codelab svc/cadence-release-cadence-web 8088:8088
```

This also runs in the foreground. Keep both port-forwards running.

**Access the Web UI:**

Open your browser and navigate to:

```
http://localhost:8088
```

You should see the Cadence Web interface. If you haven't created any domains yet, it will be mostly empty.

**Note:** To stop port-forwarding, press `Ctrl+C` in the terminal where it's running.

---

### Step 5: Create a sample domain and run a quick check

A Cadence **domain** is a namespace for workflows. Each domain has its own task lists and workflow execution history. Before running any workflows, you must register at least one domain.

This step uses `tctl` (the Cadence CLI) to register a domain. We'll exec into a frontend pod to use the tctl binary that's already installed there. Alternatively, you can [install tctl locally](https://cadenceworkflow.io/docs/cli/).

#### Register a domain

Find a frontend pod name and store it in a variable:

```bash
POD=$(kubectl get pods -n cadence-codelab \
  -l app.kubernetes.io/component=frontend \
  -o jsonpath='{.items[0].metadata.name}')
```

Register a domain called `sample-domain` with 1 day retention:

```bash
kubectl exec -n cadence-codelab -it "$POD" -- \
  tctl --address cadence-release-cadence-frontend:7833 \
  --do sample-domain domain register -rd 1
```

The `-rd 1` flag sets the workflow history retention period to 1 day. After this period, closed workflow histories may be deleted.

#### Verify domain creation

List all domains to confirm registration:

```bash
kubectl exec -n cadence-codelab -it "$POD" -- \
  tctl --address cadence-release-cadence-frontend:7833 domain list
```

You should see `sample-domain` in the output with its configuration.

#### Verify Elasticsearch integration (optional)

If you configured Elasticsearch in Step 2, Cadence will use it for advanced visibility. You can verify the Elasticsearch index was created:

**If ES is in your Kubernetes cluster:**

Port-forward to Elasticsearch (in a new terminal):

```bash
kubectl port-forward -n elastic-system svc/elasticsearch-master 9200:9200
```

Check the Cadence visibility index:

```bash
curl -s http://localhost:9200/cadence-visibility
```

**If ES is a managed service with external access:**

```bash
curl -s http://YOUR-ES-HOSTNAME:9200/cadence-visibility
```

Replace `YOUR-ES-HOSTNAME` with your actual Elasticsearch endpoint from Step 2.

You should see index metadata including mappings and settings. If you get an error, check the Elasticsearch schema job logs from Step 4.

---

### Step 6: Uninstall and cleanup (optional)

When you're done testing, you can remove Cadence and its resources from your cluster.

**Warning:** These operations will delete data. Make sure you don't need any workflow histories or domain configurations before proceeding.

#### Remove the Cadence Helm release

This removes all Cadence pods and services:

```bash
helm uninstall cadence-release -n cadence-codelab
```

This command deletes:
- All Cadence service pods (frontend, history, matching, worker, web)
- PostgreSQL database (including all workflow data)
- Kubernetes services and deployments

Note: Persistent volumes may not be automatically deleted depending on your cluster's reclaim policy.

#### Delete the namespace (optional)

If you want to remove all resources in the namespace, including any remaining persistent volume claims:

```bash
kubectl delete namespace cadence-codelab
```

This ensures complete cleanup. The namespace deletion may take a minute to complete.

#### Verify cleanup

Confirm all resources are removed:

```bash
kubectl get all -n cadence-codelab
```

You should see a "No resources found" message or an error that the namespace doesn't exist.

---

### Troubleshooting

This section covers common issues and how to resolve them.

#### Installation Issues

**Pods stuck in Pending state:**
- **Cause:** Insufficient cluster resources or quota limits
- **Solution:** Check pod status and events:
```bash
kubectl describe pod <pod-name> -n cadence-codelab
```
Look for messages about resource requests or quota exceeded. You may need to scale up your cluster or reduce resource requests in the values file.

**Helm install times out:**
- **Cause:** Schema jobs taking too long or failing
- **Solution:** Remove the `--wait` flag and monitor manually:
```bash
kubectl get pods -n cadence-codelab -w
```
Check schema job logs for specific errors.

#### Database Issues

**PostgreSQL connection failures:**
- **Cause:** Misconfigured database credentials or PostgreSQL not running
- **Solution:** Verify PostgreSQL is running:
```bash
kubectl get pods -n cadence-codelab -l app.kubernetes.io/name=postgresql
```
Confirm credentials in your values file match: `postgresql.auth.username/password/database` should match `config.persistence.database.sql.user/password/dbname`.

**Schema jobs failing:**
- **Cause:** Database connectivity or permission issues
- **Solution:** Check schema job logs:
```bash
kubectl logs job/cadence-release-cadence-schema-server -n cadence-codelab
```
Look for connection errors or SQL execution failures.

#### Elasticsearch Issues

**Elasticsearch visibility not working:**
- **Cause:** Incorrect ES configuration or unreachable ES cluster
- **Solution:** Verify ES configuration in your values file:
  - `config.persistence.elasticsearch.enabled: true`
  - `version: "v7"`
  - Correct `hosts` and `port` values
  - Valid credentials if authentication is enabled

Test connectivity to ES from within the cluster:
```bash
kubectl run -it --rm debug \
  --image=curlimages/curl \
  --restart=Never \
  -- curl -v http://YOUR-ES-HOST:9200
```

**Elasticsearch schema job failing:**
- **Cause:** Cannot reach ES or incorrect version
- **Solution:** Check ES schema job logs:
```bash
kubectl logs job/cadence-release-cadence-schema-elasticsearch -n cadence-codelab
```
Verify your ES cluster is v7 and reachable from the cluster.

#### Service Access Issues

**Cannot access Web UI at localhost:8088:**
- **Cause:** Port-forward not running or wrong port
- **Solution:** Verify port-forward is active:
```bash
kubectl port-forward -n cadence-codelab svc/cadence-release-cadence-web 8088:8088
```
Check that the web pod is running:
```bash
kubectl get pods -n cadence-codelab -l app.kubernetes.io/component=web
```

**tctl commands failing:**
- **Cause:** Frontend not accessible or wrong address
- **Solution:** Verify frontend pod is running and port-forward is active:
```bash
kubectl get pods -n cadence-codelab -l app.kubernetes.io/component=frontend
kubectl port-forward -n cadence-codelab svc/cadence-release-cadence-frontend 7833:7833
```

#### General Debugging Commands

View all resources in the namespace:
```bash
kubectl get all -n cadence-codelab
```

Check pod logs for any service:
```bash
kubectl logs -n cadence-codelab <pod-name> --tail=100
```

Describe a pod to see events and errors:
```bash
kubectl describe pod -n cadence-codelab <pod-name>
```

Check Helm release status:
```bash
helm status cadence-release -n cadence-codelab
```

### References

#### Cadence Resources

- **[Cadence Helm Charts Repository](https://github.com/cadence-workflow/cadence-charts)** - Source repository for the Helm charts used in this guide
- **[Cadence Helm Chart README](https://github.com/cadence-workflow/cadence-charts/tree/main/charts/cadence)** - Detailed chart configuration options and advanced setup scenarios
- **[Cadence CLI Documentation](https://cadenceworkflow.io/docs/cli/)** - Command-line tool reference for managing domains and workflows

#### Kubernetes and Helm

- **[Helm Installation](https://helm.sh/docs/intro/install/)** - Official guide to installing Helm on various platforms
- **[Helm Documentation](https://helm.sh/docs/)** - Complete Helm documentation for chart management
- **[kubectl Overview](https://kubernetes.io/docs/reference/kubectl/)** - Kubernetes command-line tool reference
- **[Kubernetes Documentation](https://kubernetes.io/docs/home/)** - Official Kubernetes documentation

#### Google Cloud Platform

- **[Google Cloud SDK Installation](https://cloud.google.com/sdk/docs/install)** - Install and configure gcloud CLI
- **[Creating a GKE Cluster](https://cloud.google.com/kubernetes-engine/docs/how-to/creating-a-zonal-cluster)** - Guide to creating and managing GKE clusters
- **[GKE Documentation](https://cloud.google.com/kubernetes-engine/docs)** - Complete GKE documentation

#### Database and Search

- **[PostgreSQL on Kubernetes (Bitnami)](https://artifacthub.io/packages/helm/bitnami/postgresql)** - Bitnami PostgreSQL Helm chart documentation
- **[Elasticsearch Installation](https://www.elastic.co/guide/en/elasticsearch/reference/7.17/install-elasticsearch.html)** - Elasticsearch v7 installation guide
- **[Elastic Cloud](https://www.elastic.co/cloud/)** - Managed Elasticsearch service options
