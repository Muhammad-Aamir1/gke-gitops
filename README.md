# ArgoCD Multi-Cluster GitOps Lab

A complete hands-on lab for setting up **multi-cluster GitOps** with a single management instance of ArgoCD. Keeps Dev, Staging, and Prod clusters clean, centralized, and entirely driven by Git — no direct `kubectl apply` to production.

---

## Table of Contents

- [Architectural Overview](#architectural-overview)
- [Prerequisites](#prerequisites)
- [Phase 1: Infrastructure Provisioning](#phase-1-infrastructure-provisioning)
- [Phase 2: Central ArgoCD Installation](#phase-2-central-argocd-installation)
- [Phase 3: Register Target Clusters](#phase-3-register-target-clusters)
- [Phase 4: Git Repository Structure](#phase-4-git-repository-structure)
- [Phase 5: Multi-Cluster Application Definitions](#phase-5-multi-cluster-application-definitions)
- [Phase 6: Verification](#phase-6-verification)
- [Cheat Sheet](#cheat-sheet)
- [Troubleshooting](#troubleshooting)
- [Cleanup](#cleanup)

---

## Architectural Overview

```
                  +-------------------------------+
                  |     Management Cluster        |
                  |      (ArgoCD Engine)          |
                  +---------------+---------------+   
                                  |               
            +---------------------+---------------------+
            |                     |                     |
            v                     v                     v
    +---------------+     +---------------+     +---------------+
    |  Dev Cluster  |     |Staging Cluster|     |  Prod Cluster |
    |  1 replica    |     |  2 replicas   |     |  3 replicas   |
    | nginx:alpine  |     | nginx:alpine  |     | nginx:alpine  |
    | ENV=dev       |     | ENV=staging   |     | ENV=prod      |
    +---------------+     +---------------+     +---------------+
```

**Hub and Spoke Model:**
- The **Hub** (`argocd-mgmt`) is the supervisor running ArgoCD.
- The **Spokes** (Dev, Staging, Prod) are worker clusters that receive deployments via pull-based sync.
- ArgoCD continuously compares the live state in each spoke cluster against the manifests in Git. When they drift, ArgoCD reconciles them automatically.

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| Google Cloud Platform | Active GCP project with billing enabled |
| GCP APIs | Kubernetes Engine API enabled |
| Permissions | `roles/container.admin` or equivalent |
| Tools | `gcloud`, `kubectl`, `curl` |
| GitHub Account | For hosting the manifests repository |
| Browser | For accessing the ArgoCD Web UI |

Enable required APIs:

```bash
gcloud services enable container.googleapis.com
```

---

## Phase 1: Infrastructure Provisioning

We need **4 GKE clusters**. Use **Cloud Shell** in the GCP Console.

### 1.1 Set Environment Variables

```bash
export PROJECT_ID=$(gcloud config get-value project)
export REGION=us-central1
export ZONE=us-central1-a
```

Verify your project:

```bash
echo "Project: $PROJECT_ID, Zone: $ZONE"
```

### 1.2 Create All 4 GKE Clusters

Create the management cluster (2 nodes — runs ArgoCD control plane):

```bash
gcloud container clusters create argocd-mgmt \
  --zone $ZONE \
  --num-nodes=2 \
  --machine-type=e2-standard-2 \
  --async
```

Create the 3 workload clusters (1 node each):

```bash
gcloud container clusters create gke-dev \
  --zone $ZONE \
  --num-nodes=1 \
  --machine-type=e2-standard-2 \
  --async

gcloud container clusters create gke-staging \
  --zone $ZONE \
  --num-nodes=1 \
  --machine-type=e2-standard-2 \
  --async

gcloud container clusters create gke-prod \
  --zone $ZONE \
  --num-nodes=1 \
  --machine-type=e2-standard-2 \
  --async
```

> The `--async` flag lets all 4 clusters provision simultaneously. This takes roughly 4-5 minutes.

### 1.3 Verify All Clusters Are Running

```bash
gcloud container clusters list
```

Expected output (all `STATUS: RUNNING`):

```
NAME           LOCATION       MASTER_VERSION   STATUS
argocd-mgmt    us-central1-a  X.XX.X-gke.X    RUNNING
gke-dev        us-central1-a  X.XX.X-gke.X    RUNNING
gke-staging    us-central1-a  X.XX.X-gke.X    RUNNING
gke-prod       us-central1-a  X.XX.X-gke.X    RUNNING
```

---

## Phase 2: Central ArgoCD Installation

Install the ArgoCD control plane on the `argocd-mgmt` cluster.

### 2.1 Authenticate to the Management Cluster

```bash
gcloud container clusters get-credentials argocd-mgmt --zone $ZONE
```

Verify you're on the right cluster:

```bash
kubectl config current-context
```

Expected: `gke_<PROJECT_ID>_us-central1-a_argocd-mgmt`

### 2.2 Install ArgoCD

```bash
# Create the dedicated namespace
kubectl create namespace argocd

# Apply the official non-HA installation manifest
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 2.3 Monitor the ArgoCD Pods

Wait for all pods to reach `Running`:

```bash
kubectl get pods -n argocd -w
```

Press `Ctrl+C` once all pods show `STATUS: Running`.

### 2.4 Expose the ArgoCD Server UI

Change the `argocd-server` service from `ClusterIP` to `LoadBalancer`:

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

### 2.5 Fetch the Load Balancer IP and Admin Password

Get the external IP:

```bash
kubectl get svc argocd-server -n argocd -w
```

Wait for an `EXTERNAL-IP` to appear, then press `Ctrl+C`.

Example output:

```
NAME            TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)
argocd-server   LoadBalancer   10.X.X.X       XX.XX.XX.XX      443:XXXXX/TCP
```

Get the auto-generated admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

### 2.6 Log Into the ArgoCD UI

1. Open your browser and navigate to `https://<EXTERNAL_IP>`
2. Click **Advanced > Proceed** (self-signed SSL warning is expected)
3. Log in with:
   - **Username:** `admin`
   - **Password:** (the password you just fetched)

---

## Phase 3: Register Target Clusters

ArgoCD needs explicit credentials to connect to your Dev, Staging, and Prod clusters.

### 3.1 Install the ArgoCD CLI

```bash
curl -sSL -o argocd-linux-amd64 \
  https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64

sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd

rm argocd-linux-amd64

argocd version --client
```

### 3.2 Log Into ArgoCD via CLI

```bash
argocd login <EXTERNAL_IP> \
  --username admin \
  --password <PASSWORD> \
  --insecure
```

Expected output: `'admin' logged in successfully`

### 3.3 Authenticate to All Target Clusters

```bash
gcloud container clusters get-credentials gke-dev --zone $ZONE
gcloud container clusters get-credentials gke-staging --zone $ZONE
gcloud container clusters get-credentials gke-prod --zone $ZONE
```

### 3.4 List Your Kubernetes Contexts

```bash
kubectl config get-contexts -o name
```

Expected output (example):

```
gke_<PROJECT_ID>_us-central1-a_argocd-mgmt
gke_<PROJECT_ID>_us-central1-a_gke-dev
gke_<PROJECT_ID>_us-central1-a_gke-staging
gke_<PROJECT_ID>_us-central1-a_gke-prod
```

### 3.5 Register Each Cluster with ArgoCD

Use the exact context names from the previous step:

```bash
argocd cluster add gke_<PROJECT_ID>_us-central1-a_gke-dev \
  --name dev-environment

argocd cluster add gke_<PROJECT_ID>_us-central1-a_gke-staging \
  --name staging-environment

argocd cluster add gke_<PROJECT_ID>_us-central1-a_gke-prod \
  --name prod-environment
```

Expected output for each:

```
INFO[0000] ServiceAccount "argocd-manager" created in namespace "kube-system"
INFO[0000] ClusterRole "argocd-manager-role" created
INFO[0000] ClusterRoleBinding "argocd-manager-role-binding" created
Cluster 'dev-environment' added
```

### 3.6 Verify Cluster Registration

Via CLI:

```bash
argocd cluster list
```

Expected output:

```
SERVER                          NAME
https://X.X.X.X                 dev-environment
https://X.X.X.X                 staging-environment
https://X.X.X.X                 prod-environment
https://kubernetes.default.svc  in-cluster (argocd-mgmt)
```

Via UI: Navigate to **Settings > Clusters** in ArgoCD. You should see all 3 environments.

---

## Phase 4: Git Repository Structure

This repo follows the standard GitOps folder pattern — separate directories per environment:

```
gke-gitops/
|-- dev/
|   |-- deployment.yaml       # 1 replica, env=dev
|   |-- service.yaml          # ClusterIP service
|-- staging/
|   |-- deployment.yaml       # 2 replicas, env=staging
|   |-- service.yaml          # ClusterIP service
|-- prod/
|   |-- deployment.yaml       # 3 replicas, env=prod
|   |-- service.yaml          # ClusterIP service
```

### Manifest Details

Each deployment runs **nginx:alpine** with an environment variable and label to identify the environment, plus a matching ClusterIP service.

#### dev/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
        env: dev
    spec:
      containers:
      - name: web
        image: nginx:alpine
        ports:
        - containerPort: 80
        env:
        - name: ENVIRONMENT
          value: dev
```

#### dev/service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sample-app
  namespace: default
spec:
  selector:
    app: sample-app
    env: dev
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

#### staging/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
        env: staging
    spec:
      containers:
      - name: web
        image: nginx:alpine
        ports:
        - containerPort: 80
        env:
        - name: ENVIRONMENT
          value: staging
```

#### staging/service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sample-app
  namespace: default
spec:
  selector:
    app: sample-app
    env: staging
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

#### prod/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
        env: prod
    spec:
      containers:
      - name: web
        image: nginx:alpine
        ports:
        - containerPort: 80
        env:
        - name: ENVIRONMENT
          value: prod
```

#### prod/service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sample-app
  namespace: default
spec:
  selector:
    app: sample-app
    env: prod
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

### Environment Differences

| Environment | Replicas | Label `env` | `ENVIRONMENT` Var |
|-------------|----------|-------------|-------------------|
| dev         | 1        | dev         | dev               |
| staging     | 2        | staging     | staging           |
| prod        | 3        | prod        | prod              |

> If your repo is **private**, register the Git credentials in ArgoCD under **Settings > Repositories > Connect Repo** before proceeding to Phase 5.

---

## Phase 5: Multi-Cluster Application Definitions

Instead of clicking through the UI to create 3 separate apps, we define them **declaratively** as ArgoCD `Application` CRDs.

### 5.1 Switch Back to the Management Cluster

```bash
gcloud container clusters get-credentials argocd-mgmt --zone $ZONE
```

### 5.2 Create applications.yaml

Create this file. Replace `<YOUR_GITHUB_USERNAME>` with your actual GitHub username.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: multi-cluster-app-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/<YOUR_GITHUB_USERNAME>/gke-gitops.git'
    targetRevision: HEAD
    path: dev
  destination:
    name: dev-environment
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: multi-cluster-app-staging
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/<YOUR_GITHUB_USERNAME>/gke-gitops.git'
    targetRevision: HEAD
    path: staging
  destination:
    name: staging-environment
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: multi-cluster-app-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/<YOUR_GITHUB_USERNAME>/gke-gitops.git'
    targetRevision: HEAD
    path: prod
  destination:
    name: prod-environment
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### 5.3 Apply to the Management Cluster

```bash
kubectl apply -f applications.yaml
```

Expected output:

```
application.argoproj.io/multi-cluster-app-dev created
application.argoproj.io/multi-cluster-app-staging created
application.argoproj.io/multi-cluster-app-prod created
```

### 5.4 Monitor Initial Sync

```bash
argocd app list -w
```

Press `Ctrl+C` once all apps show `SYNC_STATUS: Synced` and `HEALTH_STATUS: Healthy`.

---

## Phase 6: Verification

### 6.1 Check the ArgoCD UI Dashboard

Open the ArgoCD UI. You should see three application cards:

| Application | Sync Status | Health Status | Destination |
|-------------|-------------|---------------|-------------|
| multi-cluster-app-dev | Synced | Healthy | dev-environment |
| multi-cluster-app-staging | Synced | Healthy | staging-environment |
| multi-cluster-app-prod | Synced | Healthy | prod-environment |

Click on any application to see the live resource tree.

### 6.2 Verify from the CLI

Check the Dev cluster:

```bash
gcloud container clusters get-credentials gke-dev --zone $ZONE
kubectl get deployments,services,pods
```

Expected: 1 pod, 1 deployment, 1 service.

Check the Staging cluster:

```bash
gcloud container clusters get-credentials gke-staging --zone $ZONE
kubectl get deployments,services,pods
```

Expected: 2 pods, 1 deployment, 1 service.

Check the Prod cluster:

```bash
gcloud container clusters get-credentials gke-prod --zone $ZONE
kubectl get deployments,services,pods
```

Expected: 3 pods, 1 deployment, 1 service.

### 6.3 The Ultimate GitOps Test (Drift Reconciliation)

1. Open `prod/deployment.yaml` in your GitHub repository
2. Change `replicas: 3` to `replicas: 5`
3. Commit the change directly on GitHub (no `kubectl apply`)
4. Wait 2-3 minutes

ArgoCD polls your Git repository every 3 minutes. It detects the drift and automatically syncs to prod.

Verify:

```bash
gcloud container clusters get-credentials gke-prod --zone $ZONE
kubectl get deployments
```

Expected: `sample-app` now shows `5/5` replicas.

### 6.4 Test Self-Healing

Manually break the deployment:

```bash
kubectl scale deployment sample-app --replicas=1
```

Wait ~3 minutes. ArgoCD reverts it back to 5 replicas because `selfHeal: true`.

```bash
kubectl get deployments
```

Expected: `sample-app` back to `5/5` replicas.

---

## Cheat Sheet

```bash
# GET ARGOCD ADMIN PASSWORD
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo

# EXPOSE ARGOCD UI
kubectl patch svc argocd-server -n argocd -p \
  '{"spec": {"type": "LoadBalancer"}}'

# GET EXTERNAL IP
kubectl get svc argocd-server -n argocd -w

# REGISTER A CLUSTER
argocd cluster add <context-name> --name <env-name>

# LIST REGISTERED CLUSTERS
argocd cluster list

# LIST APPLICATIONS
argocd app list

# SYNC A SPECIFIC APP
argocd app sync multi-cluster-app-dev

# CHECK CURRENT KUBECTL CONTEXT
kubectl config current-context

# LIST ALL CONTEXTS
kubectl config get-contexts

# SWITCH CLUSTER CONTEXT
gcloud container clusters get-credentials <name> --zone $ZONE
```

---

## Troubleshooting

### Context Confusion

**Problem:** Running `kubectl` commands against the wrong cluster.

**Solution:** Always verify your current context:

```bash
kubectl config current-context
```

Switch if needed:

```bash
gcloud container clusters get-credentials argocd-mgmt --zone $ZONE
```

### Clusters Not Creating

**Problem:** `gcloud container clusters create` fails.

**Solutions:**

1. Verify billing is enabled:
   ```bash
   gcloud billing projects describe $PROJECT_ID
   ```

2. Verify the Kubernetes Engine API is enabled:
   ```bash
   gcloud services enable container.googleapis.com
   ```

3. Check quota:
   ```bash
   gcloud compute regions describe $REGION
   ```

### ArgoCD UI Not Accessible

**Problem:** Cannot reach `https://<EXTERNAL_IP>`.

**Solutions:**

1. Verify the LoadBalancer has an external IP:
   ```bash
   kubectl get svc argocd-server -n argocd
   ```

2. If `EXTERNAL-IP` shows `<pending>`, wait longer (GCP can take 1-2 minutes).

### Cluster Registration Fails

**Problem:** `argocd cluster add` returns an error.

**Solutions:**

1. Make sure you're authenticated to the target cluster:
   ```bash
   gcloud container clusters get-credentials gke-dev --zone $ZONE
   kubectl get nodes
   ```

2. Make sure you're logged into the ArgoCD CLI:
   ```bash
   argocd login <EXTERNAL_IP> --username admin --password <PASSWORD> --insecure
   ```

3. The context name is case-sensitive. Copy it exactly from:
   ```bash
   kubectl config get-contexts -o name
   ```

### Applications Not Syncing

**Problem:** ArgoCD apps show `OutOfSync` or `Missing`.

**Solutions:**

1. Check the app details for errors:
   ```bash
   argocd app get multi-cluster-app-dev
   ```

2. Verify the source repository URL is correct and accessible.

3. If using a private repo, configure credentials in **Settings > Repositories**.

4. Force a manual sync:
   ```bash
   argocd app sync multi-cluster-app-dev
   ```

### Self-Healing Not Working

**Problem:** Manual changes are not reverted.

**Solution:** Verify `selfHeal` is enabled:

```bash
kubectl get application multi-cluster-app-prod -n argocd -o yaml | grep selfHeal
```

---

## Cleanup

Delete applications:

```bash
gcloud container clusters get-credentials argocd-mgmt --zone $ZONE
kubectl delete application multi-cluster-app-dev -n argocd
kubectl delete application multi-cluster-app-staging -n argocd
kubectl delete application multi-cluster-app-prod -n argocd
```

Delete all GKE clusters:

```bash
gcloud container clusters delete argocd-mgmt --zone $ZONE --quiet
gcloud container clusters delete gke-dev --zone $ZONE --quiet
gcloud container clusters delete gke-staging --zone $ZONE --quiet
gcloud container clusters delete gke-prod --zone $ZONE --quiet
```

Verify deletion:

```bash
gcloud container clusters list
```

Expected: `Listed 0 items.`

---

## Key Concepts

| Concept | Description |
|---------|-------------|
| **GitOps** | Git is the single source of truth for infrastructure and application config |
| **Pull-Based Sync** | ArgoCD inside the cluster pulls from Git, rather than CI pushing to the cluster |
| **Application CRD** | ArgoCD custom resource defining what to deploy, where, and how to sync |
| **Self-Heal** | ArgoCD automatically reverts manual changes to match Git state |
| **Auto-Prune** | ArgoCD deletes resources in the cluster but not in Git |
| **Hub & Spoke** | Central management cluster controls multiple workload clusters |
