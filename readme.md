# Argo CD Installation Guide on Kubernetes

## Overview

This guide explains how to install **Argo CD** on a Kubernetes cluster, expose the Argo CD UI using a **LoadBalancer**, install the **Argo CD CLI**, and retrieve the initial administrator password.

---

# What is Argo CD?

Argo CD is a **GitOps Continuous Delivery** tool for Kubernetes.

Instead of manually deploying applications using `kubectl apply`, Argo CD continuously monitors a Git repository and ensures your Kubernetes cluster matches the desired state stored in Git.

### Benefits

- GitOps-based deployments
- Automated synchronization
- Rollback support
- Drift detection
- Multi-cluster management
- Web UI and CLI support
- RBAC integration

---

# Prerequisites

Before starting, ensure you have:

- Kubernetes Cluster (EKS, AKS, GKE, Minikube, Kind, etc.)
- `kubectl` configured to access the cluster
- Internet access to download manifests
- Cluster administrator permissions

Verify cluster connectivity:

```bash
kubectl get nodes
```

Expected output:

```text
NAME            STATUS   ROLES    AGE
ip-10-0-0-15    Ready    <none>   2d
```

---

# Step 1: Create the Argo CD Namespace

Namespaces help isolate Kubernetes resources.

Create a dedicated namespace for Argo CD.

```bash
kubectl create namespace argocd
```

Verify:

```bash
kubectl get ns
```

Expected:

```text
NAME       STATUS
argocd     Active
```

---

# Step 2: Install Argo CD

Deploy the official Argo CD manifests.

```bash
kubectl apply -n argocd \
-f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

This command installs:

- Argo CD API Server
- Application Controller
- Repo Server
- Redis
- Dex (Authentication)
- Notifications Controller
- CRDs
- RBAC resources

Verify pods:

```bash
kubectl get pods -n argocd
```

Example output:

```text
NAME                                      READY   STATUS
argocd-application-controller-0           1/1     Running
argocd-dex-server                         1/1     Running
argocd-notifications-controller           1/1     Running
argocd-redis                              1/1     Running
argocd-repo-server                        1/1     Running
argocd-server                             1/1     Running
```

Wait until every pod reaches the **Running** state.

---

# Step 3: Expose the Argo CD Server

By default, the Argo CD server uses a ClusterIP service, making it accessible only within the cluster.

Change the service type to **LoadBalancer** so it becomes accessible externally.

```bash
kubectl patch svc argocd-server \
-n argocd \
-p '{"spec": {"type": "LoadBalancer"}}'
```

Verify:

```bash
kubectl get svc -n argocd
```

Example:

```text
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP
argocd-server   LoadBalancer   10.100.25.10    a1b2c3.elb.amazonaws.com
```

If your cloud provider supports LoadBalancers (AWS, Azure, GCP), an external IP or DNS name will be assigned automatically.

Open:

```
https://<EXTERNAL-IP>
```

> **Note:** The browser may display a certificate warning because Argo CD uses a self-signed certificate by default.

---

# Step 4: Install the Argo CD CLI

Download the latest CLI binary.

```bash
curl -sSL -o argocd-linux-amd64 \
https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
```

Install it:

```bash
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
```

Remove the downloaded file:

```bash
rm argocd-linux-amd64
```

Verify installation:

```bash
argocd version
```

Example:

```text
argocd: v2.x.x
```

---

# Step 5: Retrieve the Initial Admin Password

Argo CD stores the initial admin password in a Kubernetes Secret.

Retrieve it using:

```bash
argocd admin initial-password -n argocd
```

Example:

```text
7Df84jkP9mnR
```

Username:

```text
admin
```

Password:

```text
<output-from-command>
```

---

# Step 6: Login Using the CLI

Replace the server address with your LoadBalancer DNS or IP.

```bash
argocd login <LOADBALANCER-IP>
```

Example:

```bash
argocd login a1b2c3.elb.amazonaws.com
```

Enter:

```text
Username: admin
Password: <initial-password>
```

---

# Step 7: Login Using the Web UI

Open:

```
https://<LOADBALANCER-IP>
```

Login with:

| Field | Value |
|--------|-------|
| Username | admin |
| Password | Initial password |

---

# Verify the Installation

Check all pods:

```bash
kubectl get pods -n argocd
```

Check services:

```bash
kubectl get svc -n argocd
```

Check deployments:

```bash
kubectl get deploy -n argocd
```

Check namespace resources:

```bash
kubectl get all -n argocd
```

---

# Common Troubleshooting

## Pods Not Running

Check pod status:

```bash
kubectl get pods -n argocd
```

Describe a pod:

```bash
kubectl describe pod <pod-name> -n argocd
```

View logs:

```bash
kubectl logs <pod-name> -n argocd
```

---

## External IP Pending

If the LoadBalancer remains in the `Pending` state:

- Verify that your Kubernetes cluster supports LoadBalancer services.
- On cloud providers (AWS, Azure, GCP), ensure the cloud controller manager is configured correctly.
- On local clusters (Minikube, Kind), consider using `kubectl port-forward` or installing MetalLB.

---

## Cannot Access UI

Check the service:

```bash
kubectl get svc -n argocd
```

Ensure the security groups or firewall rules allow HTTPS traffic (port 443).

---

## Forgot the Admin Password

Retrieve it again:

```bash
argocd admin initial-password -n argocd
```

Or inspect the secret:

```bash
kubectl get secret argocd-initial-admin-secret \
-n argocd \
-o jsonpath="{.data.password}" | base64 -d
```

---

# Useful Commands

Check pods:

```bash
kubectl get pods -n argocd
```

Check services:

```bash
kubectl get svc -n argocd
```

View logs:

```bash
kubectl logs -n argocd deployment/argocd-server
```

Restart the server:

```bash
kubectl rollout restart deployment argocd-server -n argocd
```

Restart all deployments:

```bash
kubectl rollout restart deployment -n argocd
```

---

# Architecture

```text
                 Git Repository
                       │
                       ▼
                Argo CD Repo Server
                       │
                       ▼
              Application Controller
                       │
                       ▼
               Kubernetes API Server
                       │
                       ▼
               Kubernetes Cluster
                       │
                       ▼
          Pods / Services / Deployments
```

---

# Cleanup (Optional)

Delete Argo CD completely:

```bash
kubectl delete namespace argocd
```

---

# References

- Argo CD Official Documentation: https://argo-cd.readthedocs.io/
- GitHub Repository: https://github.com/argoproj/argo-cd

---

# Summary

| Step | Description |
|------|-------------|
| 1 | Create the `argocd` namespace |
| 2 | Install Argo CD using the official manifests |
| 3 | Expose the Argo CD server as a `LoadBalancer` |
| 4 | Install the Argo CD CLI |
| 5 | Retrieve the initial admin password |
| 6 | Log in using the CLI |
| 7 | Access the Argo CD Web UI |
| 8 | Verify the installation and begin managing applications with GitOps |