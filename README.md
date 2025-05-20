**Argo CD Documentation**

---

## Table of Contents

1. [Introduction](#introduction)
2. [Prerequisites](#prerequisites)
3. [Installation](#installation)

   * [Using `kubectl`](#using-kubectl)
   * [Using Helm](#using-helm)
4. [Installing the `argocd` CLI](#installing-the-argocd-cli)
5. [Retrieving the Initial Admin Password](#retrieving-the-initial-admin-password)
6. [Creating and Managing Applications](#creating-and-managing-applications)

   * [Create an Application](#create-an-application)
   * [Sync an Application](#sync-an-application)
   * [List Projects](#list-projects)
   * [List Applications](#list-applications)
7. [Reconciliation Loop](#reconciliation-loop)

   * [Default Behavior](#default-behavior)
   * [Customize Reconciliation Interval](#customize-reconciliation-interval)
   * [Git Webhook Integration](#git-webhook-integration)
8. [Application Health Checks](#application-health-checks)
9. [Sync Strategies](#sync-strategies)
10. [Adding External Clusters](#adding-external-clusters)

---

## Introduction

Argo CD is a declarative, GitOps continuous delivery tool for Kubernetes. It continuously monitors running applications and compares the live state to the desired state defined in a Git repository. If differences are detected, Argo CD can automatically apply updates or notify operators.

## Prerequisites

* A Kubernetes cluster (v1.16+)
* `kubectl` configured to communicate with your cluster
* (Optional) Helm 3 installed if opting for the Helm-based installation

## Installation

### Using `kubectl`

Apply the stable manifest directly to your cluster:

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

This command creates the `argocd` namespace and all the necessary resources (Deployments, Services, ConfigMaps, Secrets, etc.).

### Using Helm

1. Add the Argo Helm repository:

   ```bash
   helm repo add argo https://argoproj.github.io/argo-helm
   helm repo update
   ```

2. Install the `argo-cd` chart (v4.8.0 in this example):

   ```bash
   helm install my-argo-cd argo/argo-cd --version 4.8.0 --namespace argocd --create-namespace
   ```

You can customize values by passing `--values custom-values.yaml`.

## Installing the `argocd` CLI

The `argocd` command‑line tool lets you interact with the Argo CD API server.

```bash
# Download the latest Linux AMD64 binary
tmpfile=$(mktemp)
curl -sSL -o $tmpfile https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x $tmpfile
sudo mv $tmpfile /usr/local/bin/argocd
```

Alternatively, for a specific version (e.g., v2.11.3):

```bash
wget https://github.com/argoproj/argo-cd/releases/download/v2.11.3/argocd-linux-amd64 -O argocd
chmod +x argocd
sudo mv argocd /usr/local/bin/
```

Verify installation:

```bash
argocd version
```

## Retrieving the Initial Admin Password

Argo CD generates an initial admin password stored in a Kubernetes Secret.

```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d && echo
```

Use this password to log in (username: `admin`).

## Creating and Managing Applications

### Create an Application

```bash
argocd app create <APP_NAME> \
  --repo <REPO_URL> \
  --path <APP_PATH> \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace <TARGET_NAMESPACE>
```

Example:

```bash
argocd app create guestbook \
  --repo https://github.com/argoproj/argocd-example-apps.git \
  --path guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default
```

### Sync an Application

Synchronize the live cluster state with the desired state from Git:

```bash
argocd app sync <APP_NAME>
```

### List Projects

```bash
argocd proj list
```

### List Applications

View all Argo CD Applications:

```bash
kubectl get applications -n argocd
```

## Reconciliation Loop

Argo CD periodically reconciles the live state with the desired state.

### Default Behavior

* Interval: every 3 minutes (300s)
* Controlled by the `timeout.reconciliation` setting in `argocd-cm` ConfigMap

### Customize Reconciliation Interval

1. Check current setting:

   ```bash
   kubectl describe pod argocd-repo-server -n argocd | grep -i "ARGOCD_RECONCILIATION_TIMEOUT"
   ```
2. Patch the ConfigMap:

   ```bash
   kubectl -n argocd patch configmap argocd-cm --patch '{"data":{"timeout.reconciliation":"600s"}}'
   ```
3. Restart the repo server Deployment:

   ```bash
   kubectl -n argocd rollout restart deployment argocd-repo-server
   ```

### Git Webhook Integration

To accelerate reconciliation on Git changes, configure a webhook in GitHub:

1. In your GitHub repo settings, add a webhook with the URL:

   ```text
   https://<ARGOCD_SERVER>/api/webhook
   ```
2. Select `application/json` and the push event.

Webhooks trigger near-instant syncs, reducing waiting time.

## Application Health Checks

Argo CD supports custom health checks written in Lua. You can define custom Lua scripts in the `argocd-cm` ConfigMap under the `health.lua` key.

```yaml
# Example snippet in argocd-cm:
data:
  health.lua: |
    hs = {
      healthy = {
        message = "Application is healthy"
      }
    }
    -- Your Lua script here
```

## Sync Strategies

1. **Manual Sync** – Trigger synchronization via CLI or UI.
2. **Automatic Sync** – Enable auto-sync in the Application spec.
3. **Prune Resources** – Automatically delete resources not defined in Git.
4. **Self-Healing** – Revert out-of-band changes (i.e., drift) automatically.

Example auto-sync configuration in Application YAML:

```yaml
spec:
  syncPolicy:
    automated:
      prune: true         # enable pruning
      selfHeal: true      # enable self-healing
```

## Adding External Clusters

You can manage multiple clusters from a single Argo CD instance.

1. Ensure `kubectl` is configured with contexts for all clusters.
2. Log in to Argo CD:

   ```bash
   argocd login <ARGOCD_SERVER>
   ```
3. List available contexts:

   ```bash
   kubectl config get-contexts
   ```
4. Add a cluster:

   ```bash
   argocd cluster add <CONTEXT_NAME>
   ```

This command installs the Argo CD cluster role and ServiceAccount in the target cluster and registers it with your Argo CD instance.

---

*For more details and advanced configurations, visit the [Argo CD GitHub repository](https://github.com/argoproj/argo-cd/releases).*
