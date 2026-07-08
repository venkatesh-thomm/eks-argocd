####  Argo CD Installation Guide

> A comprehensive guide to installing and configuring Argo CD on Kubernetes.

---

# Table of Contents

* [Overview](#overview)
* [What is GitOps?](#what-is-gitops)
* [Architecture](#architecture)
* [Prerequisites](#prerequisites)
* [Step 1: Create the Namespace](#step-1-create-the-namespace)
* [Step 2: Install Argo CD](#step-2-install-argo-cd)
* [Step 3: Verify the Installation](#step-3-verify-the-installation)
* [Step 4: Expose the Argo CD Server](#step-4-expose-the-argo-cd-server)
* [Step 5: Install the Argo CD CLI](#step-5-install-the-argo-cd-cli)
* [Step 6: Retrieve the Initial Admin Password](#step-6-retrieve-the-initial-admin-password)
* [Step 7: Login to Argo CD](#step-7-login-to-argo-cd)

---

# Overview

Argo CD is a **declarative GitOps Continuous Delivery (CD)** tool for Kubernetes.

Instead of manually deploying Kubernetes manifests using `kubectl apply`, Argo CD continuously monitors a Git repository and ensures that the desired state stored in Git matches the actual state of your Kubernetes cluster.

If a difference (configuration drift) is detected, Argo CD can either notify you or automatically synchronize the cluster to match the desired state.

## Key Features

* GitOps-based Continuous Delivery
* Automated synchronization
* Self-healing
* Rollback support
* Multi-cluster support
* Web UI
* CLI
* RBAC integration
* SSO support
* Deployment history and auditing

---

# What is GitOps?

GitOps is an operational framework that uses **Git as the single source of truth** for infrastructure and application deployments.

Traditional deployment workflow:

```
Developer
     │
     ▼
kubectl apply
     │
     ▼
Kubernetes Cluster
```

GitOps workflow:

```
Developer
     │
     ▼
Git Repository
     │
     ▼
Argo CD
     │
     ▼
Kubernetes Cluster
```

Whenever changes are committed to Git, Argo CD detects those changes and synchronizes the Kubernetes cluster accordingly.

---

# Architecture

```
                    Git Repository
                           │
                           │
                    Watches Repository
                           │
                           ▼
                 +--------------------+
                 |   Repo Server      |
                 +--------------------+
                           │
                           ▼
              +----------------------------+
              | Application Controller     |
              +----------------------------+
                           │
                           ▼
                  Kubernetes API Server
                           │
                           ▼
                 Kubernetes Resources
```

## Core Components

### Argo CD API Server

Provides:

* REST API
* Web UI
* CLI access
* Authentication

---

### Repository Server

Responsible for:

* Connecting to Git repositories
* Cloning repositories
* Rendering Helm charts
* Rendering Kustomize manifests

---

### Application Controller

Responsible for:

* Comparing Git state with cluster state
* Detecting drift
* Synchronizing applications
* Reporting health status

---

### Redis

Stores:

* Cache
* Session information
* Application state

---

### Dex

Provides authentication using:

* LDAP
* GitHub
* Google
* Microsoft Entra ID
* OIDC providers

---

# Prerequisites

Before installing Argo CD, ensure you have:

| Requirement           | Description                                 |
| --------------------- | ------------------------------------------- |
| Kubernetes Cluster    | EKS, AKS, GKE, OpenShift, Minikube, Kind    |
| kubectl               | Installed and configured                    |
| Cluster Admin Access  | Required to install cluster resources       |
| Internet Connectivity | Required to download installation manifests |

Verify cluster connectivity.

```bash
kubectl get nodes
```

Example:

```text
NAME           STATUS   ROLES    AGE
worker-01      Ready    <none>   5d
worker-02      Ready    <none>   5d
```

---

# Step 1: Create the Namespace

A Kubernetes namespace logically separates resources from other applications running in the cluster.

Create the `argocd` namespace:

```bash
kubectl create namespace argocd
```

Verify:

```bash
kubectl get namespace argocd
```

Expected output:

```text
NAME      STATUS   AGE
argocd    Active   10s
```

---

# Step 2: Install Argo CD

Deploy the official Argo CD installation manifests.

```bash
kubectl apply -n argocd \
-f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

This command installs the following components:

* Custom Resource Definitions (CRDs)
* Argo CD API Server
* Application Controller
* Repository Server
* Redis
* Dex
* Notifications Controller
* ConfigMaps
* Secrets
* RBAC Roles
* Service Accounts
* Services

Depending on your cluster, installation usually completes within **1–3 minutes**.

---

# Step 3: Verify the Installation

Verify that all pods are running.

```bash
kubectl get pods -n argocd
```

Expected output:

```text
NAME                                      READY   STATUS
argocd-application-controller-0           1/1     Running
argocd-applicationset-controller          1/1     Running
argocd-dex-server                         1/1     Running
argocd-notifications-controller           1/1     Running
argocd-redis                              1/1     Running
argocd-repo-server                        1/1     Running
argocd-server                             1/1     Running
```

Verify deployments:

```bash
kubectl get deployment -n argocd
```

Verify services:

```bash
kubectl get svc -n argocd
```

If any pod is not running, inspect it using:

```bash
kubectl describe pod <pod-name> -n argocd
```

Check logs:

```bash
kubectl logs <pod-name> -n argocd
```

---

# Step 4: Expose the Argo CD Server

By default, the Argo CD server is exposed internally as a `ClusterIP` service, making it inaccessible outside the Kubernetes cluster.

To access the Web UI externally, change the service type to `LoadBalancer`.

```bash
kubectl patch svc argocd-server \
-n argocd \
-p '{"spec":{"type":"LoadBalancer"}}'
```

Verify the service:

```bash
kubectl get svc -n argocd
```

Example:

```text
NAME             TYPE           CLUSTER-IP      EXTERNAL-IP
argocd-server    LoadBalancer   10.100.20.12    a12b34c56.elb.amazonaws.com
```

Wait until the `EXTERNAL-IP` is assigned.

Depending on the cloud provider, this may take **2–5 minutes**.

Once available, open:

```
https://<LOADBALANCER-IP>
```

> **Note:** The browser may display a certificate warning because Argo CD uses a self-signed TLS certificate by default. This is expected for a fresh installation.

---

# Step 5: Install the Argo CD CLI

Download the latest CLI binary:

```bash
curl -sSL -o argocd-linux-amd64 \
https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
```

Install the binary:

```bash
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
```

Remove the downloaded file:

```bash
rm argocd-linux-amd64
```

Verify the installation:

```bash
argocd version
```

Example:

```text
argocd: v3.x.x
```

---

# Step 6: Retrieve the Initial Admin Password

Argo CD stores the initial administrator password in a Kubernetes Secret.

Retrieve it using:

```bash
argocd admin initial-password -n argocd
```

Alternatively, retrieve it directly from Kubernetes:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
-o jsonpath="{.data.password}" | base64 -d && echo
```

Default username:

```
admin
```

Keep this password secure. You can change it after logging in.

---

# Step 7: Login to Argo CD

## Login Using the Web UI

Open:

```
https://<LOADBALANCER-IP>
```

Login using:

| Username | Password                                        |
| -------- | ----------------------------------------------- |
| admin    | Initial password retrieved in the previous step |

---

## Login Using the CLI

```bash
argocd login <LOADBALANCER-IP>
```

Example:

```bash
argocd login a12b34c56.elb.amazonaws.com
```

If you are using the default self-signed certificate, use:

```bash
argocd login <LOADBALANCER-IP> --insecure
```

Verify the login:

```bash
argocd account get-user-info
```

Expected output:

```text
Logged In: true
Username : admin
```

---

**Congratulations! 🎉**

You have successfully:

* Installed Argo CD
* Verified all components
* Exposed the Argo CD server
* Installed the CLI
* Retrieved the administrator password
* Logged in through both the Web UI and CLI

In the next section, you'll deploy your **first GitOps application**, learn how Argo CD synchronization works, understand the default **3-minute reconciliation interval**, configure **Auto Sync**, **Self Heal**, **Prune**, and integrate **Git webhooks** for near real-time deployments.
