# Argo CD Installation Guide

This guide walks you through installing **Argo CD** on a Kubernetes cluster, exposing the Argo CD Web UI, installing the Argo CD CLI, and logging in for the first time.

---

# Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Architecture](#architecture)
- [Step 1: Create the Namespace](#step-1-create-the-namespace)
- [Step 2: Install Argo CD](#step-2-install-argo-cd)
- [Step 3: Verify the Installation](#step-3-verify-the-installation)
- [Step 4: Expose the Argo CD Server](#step-4-expose-the-argo-cd-server)
- [Step 5: Install the Argo CD CLI](#step-5-install-the-argo-cd-cli)
- [Step 6: Retrieve the Initial Admin Password](#step-6-retrieve-the-initial-admin-password)
- [Step 7: Login to Argo CD](#step-7-login-to-argo-cd)
- [Useful Commands](#useful-commands)
- [Troubleshooting](#troubleshooting)
- [Cleanup](#cleanup)

---

# Overview

Argo CD is a **GitOps Continuous Delivery** tool for Kubernetes.

Instead of manually deploying applications using `kubectl apply`, Argo CD continuously compares the desired state stored in a Git repository with the actual state of your Kubernetes cluster.

If any differences (configuration drift) are detected, Argo CD can:

- Automatically synchronize changes
- Notify users of drift
- Roll back to previous versions
- Provide deployment history
- Visualize application health

## Key Features

- GitOps-based deployments
- Automated synchronization
- Self-healing applications
- Rollback support
- Multi-cluster management
- Web UI
- CLI
- RBAC support
- SSO integration

---

# Prerequisites

Before installing Argo CD, ensure you have the following:

| Requirement | Description |
|------------|-------------|
| Kubernetes Cluster | EKS, AKS, GKE, OpenShift, Minikube, Kind, etc. |
| kubectl | Configured to communicate with the cluster |
| Cluster Admin Access | Required for installation |
| Internet Access | Required to download the installation manifests |

Verify cluster connectivity:

```bash
kubectl get nodes
```

Expected output:

```text
NAME            STATUS   ROLES    AGE
worker-node-1   Ready    <none>   12d
worker-node-2   Ready    <none>   12d
```

---

# Architecture

```
                 Git Repository
                        │
                        │
              Watches Git Repository
                        │
                        ▼
                +----------------+
                |    Repo Server |
                +----------------+
                        │
                        ▼
              +----------------------+
              | Application Controller|
              +----------------------+
                        │
                        ▼
               Kubernetes API Server
                        │
                        ▼
              Kubernetes Resources
```

---

# Step 1: Create the Namespace

Namespaces logically separate Kubernetes resources.

Create a dedicated namespace for Argo CD.

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
argocd    Active   15s
```

---

# Step 2: Install Argo CD

Deploy the official Argo CD manifests.

```bash
kubectl apply -n argocd \
-f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

This command installs:

- Custom Resource Definitions (CRDs)
- Argo CD API Server
- Repository Server
- Application Controller
- Redis
- Dex Authentication Server
- Notifications Controller
- RBAC Resources
- Service Accounts
- Services
- ConfigMaps
- Secrets

Installation usually completes within 1–3 minutes depending on the cluster.

---

# Step 3: Verify the Installation

Verify that all pods are running.

```bash
kubectl get pods -n argocd
```

Example:

```text
NAME                                      READY   STATUS
argocd-application-controller-0           1/1     Running
argocd-dex-server                         1/1     Running
argocd-notifications-controller           1/1     Running
argocd-redis                              1/1     Running
argocd-repo-server                        1/1     Running
argocd-server                             1/1     Running
```

If any pod is not in the `Running` state, inspect it using:

```bash
kubectl describe pod <pod-name> -n argocd
```

---

# Step 4: Expose the Argo CD Server

By default, the Argo CD API server is exposed as a **ClusterIP** service, meaning it is only accessible from within the Kubernetes cluster.

To access the Argo CD Web UI externally, change the service type to **LoadBalancer**.

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
NAME             TYPE           EXTERNAL-IP
argocd-server    LoadBalancer   a12345.elb.amazonaws.com
```

Open the following URL in your browser:

```
https://<EXTERNAL-IP>
```

> **Note:** The browser may display a security warning because Argo CD uses a self-signed TLS certificate by default. This is expected for a fresh installation.

---

# Step 5: Install the Argo CD CLI

Download the latest CLI binary.

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

During installation, Argo CD creates an initial administrator password and stores it in a Kubernetes Secret.

Retrieve it using:

```bash
argocd admin initial-password -n argocd
```

Alternatively, you can retrieve it directly from the Kubernetes Secret:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
-o jsonpath="{.data.password}" | base64 -d && echo
```

Default username:

```text
admin
```

---

# Step 7: Login to Argo CD

## Login Using the Web UI

Navigate to:

```
https://<LOADBALANCER-IP>
```

Use:

| Username | Password |
|-----------|----------|
| admin | Initial password retrieved above |

---

## Login Using the CLI

```bash
argocd login <LOADBALANCER-IP>
```

Example:

```bash
argocd login a12345.elb.amazonaws.com
```

If you are using the default self-signed certificate, you may need to skip certificate verification during the initial login:

```bash
argocd login <LOADBALANCER-IP> --insecure
```

---

# Useful Commands

## View Pods

```bash
kubectl get pods -n argocd
```

## View Services

```bash
kubectl get svc -n argocd
```

## View Deployments

```bash
kubectl get deployment -n argocd
```

## View All Resources

```bash
kubectl get all -n argocd
```

## Check Logs

```bash
kubectl logs deployment/argocd-server -n argocd
```

## Restart the Argo CD Server

```bash
kubectl rollout restart deployment argocd-server -n argocd
```

---

# Troubleshooting

## Pods Are Not Running

Check pod status:

```bash
kubectl get pods -n argocd
```

Describe the pod:

```bash
kubectl describe pod <pod-name> -n argocd
```

View logs:

```bash
kubectl logs <pod-name> -n argocd
```

---

## External IP Is Pending

If the `EXTERNAL-IP` remains in the `Pending` state:

- Verify your Kubernetes cluster supports `LoadBalancer` services.
- Ensure your cloud provider's load balancer integration is enabled.
- For local clusters (Kind, Minikube), use `kubectl port-forward` or install MetalLB.

---

## Cannot Access the UI

Check that the service has an external IP:

```bash
kubectl get svc -n argocd
```

Ensure:

- Port **443** is open in your firewall or security group.
- DNS resolves correctly if using a hostname.
- The Argo CD server pod is healthy.

---

## Forgot the Admin Password

Retrieve it again:

```bash
argocd admin initial-password -n argocd
```

Or reset the admin password by following the official Argo CD documentation.

---

# Cleanup

To remove Argo CD from the cluster:

```bash
kubectl delete namespace argocd
```

This deletes all Argo CD resources, including deployments, services, secrets, and CRDs within the namespace.

---

# References

- Official Documentation: https://argo-cd.readthedocs.io/
- GitHub Repository: https://github.com/argoproj/argo-cd

---

# Summary

| Step | Description |
|------|-------------|
| 1 | Create the `argocd` namespace |
| 2 | Install Argo CD using the official manifests |
| 3 | Verify all Argo CD components are running |
| 4 | Expose the Argo CD server via a `LoadBalancer` |
| 5 | Install the Argo CD CLI |
| 6 | Retrieve the initial administrator password |
| 7 | Log in to the Argo CD Web UI or CLI |