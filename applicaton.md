#### Deploying Your First Application and Understanding Synchronization

After installing Argo CD, the next step is to deploy and manage applications using GitOps.

In this section, you will learn:

* How Argo CD applications work
* Create your first application
* Manual synchronization
* Automatic synchronization
* Self-healing
* Pruning resources
* Default synchronization interval
* Git webhooks
* GitOps deployment workflow

---

# Understanding an Argo CD Application

An **Application** is the core resource managed by Argo CD.

It defines:

* Where your application source code or Kubernetes manifests are stored (Git repository)
* Which branch, tag, or commit to use
* Which folder contains the manifests
* Which Kubernetes cluster to deploy to
* Which namespace to deploy into
* How synchronization should behave

Think of it as a deployment blueprint.

---

# Deploy Your First Application

For this example, we'll deploy the guestbook application maintained by the Argo CD project.

Create the application:

```bash
argocd app create guestbook \
  --repo https://github.com/argoproj/argocd-example-apps.git \
  --path guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default
```

### Command Explanation

| Option             | Description                                                  |
| ------------------ | ------------------------------------------------------------ |
| `--repo`           | Git repository containing Kubernetes manifests               |
| `--path`           | Folder within the repository                                 |
| `--dest-server`    | Kubernetes API server where the application will be deployed |
| `--dest-namespace` | Target Kubernetes namespace                                  |

Verify the application:

```bash
argocd app list
```

Expected output:

```text
NAME        STATUS      HEALTH
guestbook   OutOfSync   Missing
```

At this stage, the application exists in Argo CD but has not yet been deployed to the cluster.

---

# Synchronize the Application

To deploy the application, synchronize it manually.

```bash
argocd app sync guestbook
```

Verify the application status:

```bash
argocd app get guestbook
```

Expected output:

```text
Sync Status: Synced
Health Status: Healthy
```

You can also view the deployment in Kubernetes:

```bash
kubectl get all -n default
```

---

# Manual Synchronization

Manual synchronization gives you complete control over when changes are deployed.

Typical workflow:

```
Developer
    │
    ▼
Commit Changes
    │
    ▼
Git Repository
    │
    ▼
Argo CD
    │
    ▼
OutOfSync
    │
Manual Approval
    │
    ▼
SYNC
    │
    ▼
Kubernetes Cluster
```

Use manual synchronization when:

* Deploying production workloads
* Following change approval processes
* Requiring validation before deployment

Synchronize manually:

```bash
argocd app sync guestbook
```

---

# Automatic Synchronization

Instead of manually clicking **Sync**, Argo CD can automatically deploy changes whenever the Git repository is updated.

Enable Auto Sync:

```bash
argocd app set guestbook --sync-policy automated
```

Now the workflow becomes:

```
Developer
    │
    ▼
Push Code
    │
    ▼
Git Repository
    │
    ▼
Argo CD Detects Change
    │
    ▼
Automatic Deployment
    │
    ▼
Kubernetes Cluster
```

No manual intervention is required after changes are committed.

---

# Self-Healing

Configuration drift occurs when someone modifies Kubernetes resources directly using `kubectl edit`, `kubectl apply`, or another tool.

With **Self-Heal** enabled, Argo CD automatically restores the resources to match the desired state stored in Git.

Enable Self-Heal:

```bash
argocd app set guestbook --self-heal
```

Example:

1. A Deployment is manually scaled to 10 replicas.
2. Git specifies 3 replicas.
3. Argo CD detects the drift.
4. Argo CD automatically changes the Deployment back to 3 replicas.

This ensures Git remains the single source of truth.

---

# Prune

Suppose a Deployment is deleted from your Git repository.

Without pruning:

* The Deployment remains running in Kubernetes.

With pruning enabled:

* Argo CD removes the Deployment from the cluster during synchronization.

Enable pruning:

```bash
argocd app set guestbook --auto-prune
```

This helps prevent orphaned resources from accumulating in the cluster.

---

# Recommended Sync Policy

A typical production sync policy looks like this:

```yaml
spec:
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### Explanation

| Option           | Purpose                                    |
| ---------------- | ------------------------------------------ |
| `automated`      | Automatically deploys changes from Git     |
| `prune: true`    | Removes resources deleted from Git         |
| `selfHeal: true` | Reverts manual changes made in the cluster |

---

# How Often Does Argo CD Check Git?

By default, Argo CD checks the Git repository approximately every **3 minutes (180 seconds)**.

The reconciliation process:

```
Git Repository
      │
      │ Every ~3 minutes
      ▼
Argo CD
      │
Compare Desired State
      │
      ▼
Detect Drift
      │
      ▼
Synchronize Cluster
```

> **Note:** Argo CD adds a small random delay (jitter) to the polling interval so that multiple applications do not poll Git at exactly the same time.

---

# Changing the Reconciliation Interval

The polling interval can be changed in the `argocd-cm` ConfigMap.

View the current configuration:

```bash
kubectl get configmap argocd-cm -n argocd -o yaml
```

Example:

```yaml
data:
  timeout.reconciliation: 180s
```

To change the interval to 60 seconds:

```yaml
data:
  timeout.reconciliation: 60s
```

Apply the updated ConfigMap:

```bash
kubectl apply -f argocd-cm.yaml
```

> **Recommendation:** Keep the default value unless you have a specific requirement. Lower intervals increase load on both Argo CD and your Git server.

---

# Git Webhooks (Recommended)

Polling every 3 minutes works well, but changes can be deployed even faster using Git webhooks.

Workflow:

```
Developer
    │
    ▼
Push Code
    │
    ▼
GitHub / GitLab
    │
Webhook Notification
    ▼
Argo CD
    │
Immediate Refresh
    ▼
Kubernetes Cluster
```

Benefits of webhooks:

* Near real-time deployments
* Reduced Git polling
* Lower latency
* Faster feedback

Most Git providers (GitHub, GitLab, Bitbucket, Azure DevOps) support webhooks.

---

# Check Application Status

View all applications:

```bash
argocd app list
```

View application details:

```bash
argocd app get guestbook
```

Check synchronization history:

```bash
argocd app history guestbook
```

View application logs:

```bash
argocd app logs guestbook
```

---

# Refresh an Application

To force Argo CD to immediately refresh the application state from Git:

```bash
argocd app get guestbook --refresh
```

This is useful when you don't want to wait for the next reconciliation cycle.

---

# Suspend Automatic Synchronization

If you need to temporarily stop automatic deployments:

```bash
argocd app set guestbook --sync-policy none
```

Manual synchronization is still available using:

```bash
argocd app sync guestbook
```

---

# GitOps Workflow Summary

```
Developer
    │
    ▼
Commit Code
    │
    ▼
Push to Git
    │
    ▼
Git Repository
    │
    ├── Poll every ~3 minutes
    │         OR
    └── Webhook Notification
              │
              ▼
          Argo CD
              │
      Compare Desired State
              │
              ▼
      Detect Configuration Drift
              │
              ▼
      Synchronize Kubernetes Cluster
              │
              ▼
      Application Becomes Healthy
```

---

# Best Practices

* Store all Kubernetes manifests in Git.
* Enable `selfHeal` to prevent configuration drift.
* Enable `prune` to remove obsolete resources.
* Use Git webhooks for faster synchronization.
* Protect production branches with pull request approvals.
* Monitor application health and synchronization status regularly.
* Avoid making manual changes directly in the cluster, as they may be overwritten by Argo CD when self-healing is enabled.
