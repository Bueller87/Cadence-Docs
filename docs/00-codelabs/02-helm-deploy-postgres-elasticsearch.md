---
layout: default 
title: Deploy Cadence with Helm on GKE
permalink: /docs/codelabs/helm-deploy-postgres-elasticsearch
---
## Codelab: Deploy Cadence with Helm on GKE

Ready to deploy Cadence? This codelab walks you through a complete, production-ready deployment using official Helm charts—no custom scripting required. Whether you're new to Kubernetes or an experienced platform engineer, you'll have Cadence running on GKE with PostgreSQL, Kafka, and Elasticsearch in under 30 minutes.

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
- Kafka for visibility event streaming
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

For this guide, we'll assume your cluster is named `cadence-test-gke-1`. Replace this with your actual cluster name in the commands below.

#### Connect kubectl to your cluster

Get credentials for your existing GKE cluster (replace `cadence-test-gke-1` with your cluster name):

```bash
gcloud container clusters get-credentials cadence-test-gke-1
```

**Note:** If this command fails (404, not found), you may need to specify the region or zone where your cluster is located:

```bash
gcloud container clusters get-credentials cadence-test-gke-1 --region us-central1
```

Verify the connection by listing nodes:

```bash
kubectl get nodes
```

You should see a list of nodes in your cluster with a `Ready` status.

#### Create a namespace for Cadence

Namespaces provide logical isolation within the cluster:

```bash
kubectl create namespace cadence-postgres-es7
```

Verify the namespace was created:

```bash
kubectl get namespace cadence-postgres-es7
```

---

### Step 2: Use the official Helm values file for PostgreSQL + Elasticsearch v7 + Kafka

This step uses the official example values file that configures Cadence with PostgreSQL as the main database, Kafka for visibility event streaming, and Elasticsearch v7 for advanced visibility features.

#### Deployment architecture

This guide deploys a complete visibility stack as part of the same Helm release:
- **PostgreSQL** - Main persistence store for workflow data
- **Kafka** - Event streaming for visibility events (required for Elasticsearch integration)
- **Elasticsearch v7** - Advanced visibility search and filtering
- **Cadence services** - Frontend, history, matching, worker

The Cadence Helm chart includes subcharts for all dependencies, so everything is deployed and configured together automatically.

**Benefits of this approach:**
- Simple setup: Everything deployed together in one command
- Automatic configuration: Service discovery and connectivity handled automatically
- Production-ready architecture: Real event streaming with Kafka
- Good for development, testing, and production environments

**For production environments,** you may prefer using managed services (Cloud SQL, Elastic Cloud, Confluent Cloud) or separate deployments. The values file can be easily modified to point to external services instead.

#### View the official values file

The `cadence-charts` repository includes a tested example file for this exact deployment. You can view it:

```bash
cat charts/cadence/examples/values.postgres-es7.yaml
```

This file is maintained by the Cadence team and includes all necessary configuration for PostgreSQL, Kafka (KRaft mode), and Elasticsearch with GKE Autopilot compatibility.

#### What this configuration does

The official values file configures a complete Cadence deployment with three stateful services: PostgreSQL, Kafka, and Elasticsearch. Here's what each component does:

**Global Security Setting:**
- `allowInsecureImages: true` - Required to use images from the `bitnamilegacy` repository
- Bitnami charts validate that images come from approved repositories; legacy images are flagged as "unrecognized"
- This setting bypasses the validation check

**Image Repository Note:**
- This configuration uses the `bitnamilegacy` repository for PostgreSQL, Kafka, and Elasticsearch images
- **Why?** In August 2025, Bitnami transitioned free container images to a legacy repository. These images are stable and working, but won't receive updates
- For production deployments, consider:
  - Using managed services (Cloud SQL, Confluent Cloud, Elastic Cloud)
  - Hosting your own registry with vetted images
  - Exploring Bitnami's secure images (requires subscription)
- **Finding alternative images:** Check [Docker Hub](https://hub.docker.com) for `bitnamilegacy/postgresql`, `bitnamilegacy/kafka`, and `bitnamilegacy/elasticsearch` tags

**Database (PostgreSQL):**
- Deploys PostgreSQL 16.4 from the Bitnami legacy repository
- Creates two databases: `cadence` (main) and `cadence_visibility`
- PostgreSQL hostname: `cadence-release-postgresql.cadence-postgres-es7.svc.cluster.local`
- 8Gi persistent volume for data storage
- Suitable for development, testing, and production

**Kafka (Event Streaming):**
- **Required for Elasticsearch integration** - Cadence streams visibility events to Kafka, which Elasticsearch consumes
- Deploys Kafka 3.8.0 in **KRaft mode** (Kafka Raft - no ZooKeeper required)
- Single controller and single broker configuration (suitable for development and testing)
- Kafka hostname: `cadence-release-kafka.cadence-postgres-es7.svc.cluster.local`
- 8Gi persistent volumes for both controller and broker
- **GKE Autopilot compatibility:**
  - `sysctlImage.enabled: false` - Disables privileged init container
  - Security contexts configured to run as non-root user (UID 1001)
  - Explicit CPU/memory resource requests and limits
- **KRaft mode benefits:**
  - Simplified architecture without ZooKeeper dependency
  - Faster startup and better performance
  - Kafka's modern, recommended deployment mode

**Elasticsearch (Advanced Visibility):**
- Deploys Elasticsearch v7.17.23 from the Bitnami legacy repository
- **Version 7 is required** - Cadence does not support Elasticsearch v8 yet
- Configured with a single master node that handles all roles (suitable for development and testing)
- Elasticsearch hostname: `cadence-release-elasticsearch.cadence-postgres-es7.svc.cluster.local`
- Enables advanced workflow search and filtering capabilities
- 8Gi persistent volume for indices
- **GKE Autopilot compatibility:**
  - `sysctlImage.enabled: false` - Disables privileged init container
  - Explicit CPU/memory resource requests (1 CPU, 1024Mi) and limits (2 CPU, 2048Mi)
  - Security contexts for non-root execution

**Visibility Architecture:**
- Cadence workflows generate visibility events
- Events are streamed to Kafka topics
- Elasticsearch consumes events from Kafka and indexes them
- The `dynamicConfig` section configures Cadence to **write to and read from Elasticsearch** (`es-visibility`)
- This enables advanced workflow queries like filtering by custom fields, full-text search, and complex date ranges

**Schema Jobs:**
- Automatically runs database schema setup for PostgreSQL
- Automatically runs index setup for Elasticsearch
- Jobs run once during installation and complete

**Cadence Web UI:**
- Deploys the web interface as a ClusterIP service
- Accessible via port-forward (shown in later steps)

---

### Step 3: Install Cadence with Helm

This step installs Cadence and its dependencies (PostgreSQL, Kafka, and Elasticsearch) into your cluster using the official values file.

#### Add Helm repositories

First, ensure you're in the `cadence-charts` directory:

```bash
cd cadence-charts
```

Add the Bitnami repository (needed for the PostgreSQL, Kafka, and Elasticsearch subcharts):

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```

Update Helm repositories to get the latest chart versions:

```bash
helm repo update
```

Build chart dependencies to download the PostgreSQL, Kafka, and Elasticsearch subcharts:

```bash
helm dependency build ./charts/cadence
```

This command downloads all required dependency charts into `charts/cadence/charts/`. You'll see output showing each subchart being downloaded and saved.

#### Install Cadence

Install Cadence using the official values file from Step 2. This command will:
- Create a Helm release named `cadence-release`
- Deploy to the `cadence-postgres-es7` namespace
- Install PostgreSQL, Kafka, Elasticsearch, Cadence services (frontend, history, matching, worker), and Cadence Web
- Run schema initialization jobs for both PostgreSQL and Elasticsearch
- Wait until all resources are ready (typically 5-10 minutes with Kafka and Elasticsearch)

```bash
helm upgrade --install cadence-release ./charts/cadence \
  --namespace cadence-postgres-es7 \
  -f charts/cadence/examples/values.postgres-es7.yaml \
  --wait
```

The `--wait` flag keeps the command running until all pods are ready. You'll see status messages as resources are created.

**Note:** If using GKE Autopilot, you'll see warnings about "autopilot-default-resources-mutator" defaulting CPU resources. This is normal Autopilot behavior and not an error - Autopilot automatically adds resource requests to containers that don't specify them.

#### Verify installation

Check that all pods are running:

```bash
kubectl get pods -n cadence-postgres-es7
```

You should see pods for:
- `cadence-release-frontend`
- `cadence-release-history`
- `cadence-release-matching`
- `cadence-release-worker`
- `cadence-release-web`
- `cadence-release-postgresql`
- `cadence-release-kafka-controller`
- `cadence-release-kafka-broker`
- `cadence-release-elasticsearch-master`
- Schema jobs (may show as `Completed` or already gone)

All pods should show `Running` status (or `Completed` for jobs).

Check services:

```bash
kubectl get svc -n cadence-postgres-es7
```

You should see services for frontend, web, PostgreSQL, Kafka, and Elasticsearch.

---

### Step 4: Verify schema jobs and Cadence health

This step confirms that database schemas were initialized correctly and sets up local access to Cadence services.

#### Check schema jobs

Schema jobs run once during installation to set up database tables and Elasticsearch indices. Check if the jobs completed successfully:

```bash
kubectl get jobs -n cadence-postgres-es7
```

You should see jobs with `COMPLETIONS` showing `1/1`. If jobs are still present, check their logs:

View PostgreSQL schema job logs:

```bash
kubectl logs job/cadence-release-schema-postgresql -n cadence-postgres-es7 --tail=200
```

View Elasticsearch schema job logs:

```bash
kubectl logs job/cadence-release-schema-elasticsearch -n cadence-postgres-es7 --tail=200
```

Look for messages indicating successful schema creation. If you see errors, they typically relate to connectivity issues with PostgreSQL or Elasticsearch.

#### Access Cadence services locally

Port-forwarding creates a tunnel from your local machine to services running in the cluster. This allows you to interact with Cadence as if it were running locally.

**Port-forward the Cadence frontend** (for CLI access):

```bash
kubectl port-forward -n cadence-postgres-es7 svc/cadence-release-frontend 7833:7833
```

This command runs in the foreground. Keep it running and open a new terminal for the next command.

**In a new terminal, port-forward the Cadence Web UI:**

```bash
kubectl port-forward -n cadence-postgres-es7 svc/cadence-release-web 8088:8088
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

This step uses the Cadence CLI to register a domain. We'll exec into a frontend pod to use the `cadence` CLI binary that's already installed there. Alternatively, you can [install the Cadence CLI locally](https://cadenceworkflow.io/docs/cli/).

#### Register a domain

Find a frontend pod name and store it in a variable:

```bash
POD=$(kubectl get pods -n cadence-postgres-es7 \
  -l app.kubernetes.io/component=frontend \
  -o jsonpath='{.items[0].metadata.name}')
```

Register a domain called `sample-domain` with 1 day retention:

```bash
kubectl exec -n cadence-postgres-es7 -it "$POD" -- \
  cadence --address cadence-release-frontend:7833 \
  --do sample-domain domain register -rd 1
```

The `-rd 1` flag sets the workflow history retention period to 1 day. After this period, closed workflow histories may be deleted.

#### Verify domain creation

List all domains to confirm registration:

```bash
kubectl exec -n cadence-postgres-es7 -it "$POD" -- \
  cadence --address cadence-release-frontend:7833 domain list
```

You should see `sample-domain` in the output with its configuration.

#### Verify Kafka integration (optional)

Kafka is required for streaming visibility events to Elasticsearch. You can verify Kafka pods are running:

Check Kafka pod status:

```bash
kubectl get pods -n cadence-postgres-es7 -l app.kubernetes.io/name=kafka
```

You should see both `cadence-release-kafka-controller` and `cadence-release-kafka-broker` in `Running` status.

View Kafka controller logs to verify KRaft initialization:

```bash
kubectl logs -n cadence-postgres-es7 -l app.kubernetes.io/component=controller --tail=50
```

Look for messages indicating successful KRaft cluster formation and controller election.

#### Verify Elasticsearch integration (optional)

Cadence is configured to use Elasticsearch for advanced visibility. You can verify the Elasticsearch index was created:

Port-forward to Elasticsearch (in a new terminal):

```bash
kubectl port-forward -n cadence-postgres-es7 svc/cadence-release-elasticsearch 9200:9200
```

Check the Cadence visibility index:

```bash
curl -s http://localhost:9200/cadence-visibility
```

You should see index metadata including mappings and settings. If you get an error, check the Elasticsearch schema job logs from earlier in this step.

---

### Step 6: Uninstall and cleanup (optional)

When you're done testing, you can remove Cadence and its resources from your cluster.

**Warning:** These operations will delete data. Make sure you don't need any workflow histories or domain configurations before proceeding.

Choose one of the following cleanup options:

#### Option A: Remove only Cadence (keep the namespace)

Use this if you want to keep the namespace for other workloads:

```bash
helm uninstall cadence-release -n cadence-postgres-es7
```

This command deletes:
- All Cadence service pods (frontend, history, matching, worker, web)
- PostgreSQL database (including all workflow data)
- Kafka cluster (controller and broker with all event data)
- Elasticsearch cluster (including all visibility data)
- Kubernetes services and deployments

Note: Persistent volumes may not be automatically deleted depending on your cluster's reclaim policy.

#### Option B: Complete cleanup (delete the namespace)

Use this for complete removal of everything:

```bash
kubectl delete namespace cadence-postgres-es7
```

This deletes:
- Everything from Option A
- All remaining persistent volume claims and their data (PostgreSQL, Kafka controller, Kafka broker, Elasticsearch)
- The namespace itself

**Note:** If using Option B, you don't need to run `helm uninstall` first. Deleting the namespace removes everything including the Helm release.

The namespace deletion may take a minute to complete.

#### Verify cleanup

Confirm all resources are removed:

```bash
kubectl get all -n cadence-postgres-es7
```

You should see a "No resources found" message or an error that the namespace doesn't exist.

---

### Troubleshooting

This section covers common issues and how to resolve them.

#### Installation Issues

**Image pull errors (ImagePullBackOff):**
- **Cause:** Docker image tag doesn't exist or repository access issues
- **Solution:** Verify the image tags in your values file match available images in the `bitnamilegacy` repository
- Check [Docker Hub bitnamilegacy/postgresql](https://hub.docker.com/r/bitnamilegacy/postgresql/tags) for PostgreSQL tags
- Check [Docker Hub bitnamilegacy/kafka](https://hub.docker.com/r/bitnamilegacy/kafka/tags) for Kafka tags
- Check [Docker Hub bitnamilegacy/elasticsearch](https://hub.docker.com/r/bitnamilegacy/elasticsearch/tags) for Elasticsearch tags
- If images are unavailable, you may need to use alternative repositories or managed services

**Pods stuck in Pending state:**
- **Cause:** Insufficient cluster resources or quota limits
- **Solution:** Check pod status and events:
```bash
kubectl describe pod <pod-name> -n cadence-postgres-es7
```
Look for messages about resource requests or quota exceeded. You may need to scale up your cluster or reduce resource requests in the values file.

**Helm install times out:**
- **Cause:** Schema jobs taking too long or failing
- **Solution:** Remove the `--wait` flag and monitor manually:
```bash
kubectl get pods -n cadence-postgres-es7 -w
```
Check schema job logs for specific errors.

#### Database Issues

**PostgreSQL connection failures:**
- **Cause:** Misconfigured database credentials or PostgreSQL not running
- **Solution:** Verify PostgreSQL is running:
```bash
kubectl get pods -n cadence-postgres-es7 -l app.kubernetes.io/name=postgresql
```
Confirm credentials in your values file match: `postgresql.auth.username/password/database` should match `config.persistence.database.sql.user/password/dbname`.

**Schema jobs failing:**
- **Cause:** Database connectivity or permission issues
- **Solution:** Check schema job logs:
```bash
kubectl logs job/cadence-release-schema-server -n cadence-postgres-es7
```
Look for connection errors or SQL execution failures.

#### Elasticsearch Issues

**Elasticsearch visibility not working:**
- **Cause:** Elasticsearch pod not running or unreachable
- **Solution:** Verify Elasticsearch pod is running:
```bash
kubectl get pods -n cadence-postgres-es7 -l app.kubernetes.io/name=elasticsearch
```

Test connectivity to ES from within the cluster:
```bash
kubectl run -it --rm debug \
  --image=curlimages/curl \
  --restart=Never \
  -- curl -v http://cadence-release-elasticsearch-master.cadence-postgres-es7.svc.cluster.local:9200
```

**Elasticsearch schema job failing:**
- **Cause:** Cannot reach ES or incorrect version
- **Solution:** Check ES schema job logs:
```bash
kubectl logs job/cadence-release-schema-elasticsearch -n cadence-postgres-es7
```
Verify your ES cluster is v7 and reachable from the cluster.

#### Kafka Issues

**Kafka pods not starting:**
- **Cause:** Resource constraints, volume issues, or KRaft initialization problems
- **Solution:** Check Kafka pod status:
```bash
kubectl get pods -n cadence-postgres-es7 -l app.kubernetes.io/name=kafka
```
Check controller pod logs for errors:
```bash
kubectl logs -n cadence-postgres-es7 -l app.kubernetes.io/component=controller --tail=100
```
Check broker pod logs:
```bash
kubectl logs -n cadence-postgres-es7 -l app.kubernetes.io/component=broker --tail=100
```

**KRaft cluster formation issues:**
- **Cause:** Controller initialization failures or misconfiguration
- **Solution:** Look for KRaft-related errors in controller logs. Common issues include:
  - Metadata directory permissions (should be writable by UID 1001)
  - Insufficient resources (Kafka needs at least 500m CPU and 1Gi memory)
  - Network connectivity issues between controller and broker

**Kafka connectivity from Cadence:**
- **Cause:** Service name resolution or network policy issues
- **Solution:** Verify Kafka service is accessible:
```bash
kubectl get svc -n cadence-postgres-es7 -l app.kubernetes.io/name=kafka
```
Expected service name: `cadence-release-kafka.cadence-postgres-es7.svc.cluster.local:9092`

Test connectivity from within cluster:
```bash
kubectl run -it --rm debug \
  --image=busybox \
  --restart=Never \
  -- telnet cadence-release-kafka.cadence-postgres-es7.svc.cluster.local 9092
```

**Kafka volume permission errors:**
- **Cause:** GKE Autopilot security contexts or PVC issues
- **Solution:** Verify security contexts are set correctly (runAsUser: 1001, fsGroup: 1001). Check PVC status:
```bash
kubectl get pvc -n cadence-postgres-es7 -l app.kubernetes.io/name=kafka
```
All PVCs should be `Bound`.

#### Service Access Issues

**Cannot access Web UI at localhost:8088:**
- **Cause:** Port-forward not running or wrong port
- **Solution:** Verify port-forward is active:
```bash
kubectl port-forward -n cadence-postgres-es7 svc/cadence-release-web 8088:8088
```
Check that the web pod is running:
```bash
kubectl get pods -n cadence-postgres-es7 -l app.kubernetes.io/component=web
```

**cadence CLI commands failing:**
- **Cause:** Frontend not accessible or wrong address
- **Solution:** Verify frontend pod is running and port-forward is active:
```bash
kubectl get pods -n cadence-postgres-es7 -l app.kubernetes.io/component=frontend
kubectl port-forward -n cadence-postgres-es7 svc/cadence-release-frontend 7833:7833
```

#### General Debugging Commands

View all resources in the namespace:
```bash
kubectl get all -n cadence-postgres-es7
```

Check pod logs for any service:
```bash
kubectl logs -n cadence-postgres-es7 <pod-name> --tail=100
```

Describe a pod to see events and errors:
```bash
kubectl describe pod -n cadence-postgres-es7 <pod-name>
```

Check Helm release status:
```bash
helm status cadence-release -n cadence-postgres-es7
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
