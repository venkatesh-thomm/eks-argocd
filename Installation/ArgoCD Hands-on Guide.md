#### Operations, Troubleshooting, Best Practices & Cheat Sheet


# Deleting an Application

Deleting an application from Argo CD removes the Application resource. Whether Kubernetes resources are also deleted depends on the deletion policy.

Delete an application:

```bash
argocd app delete guestbook
```

To cascade the deletion and remove Kubernetes resources managed by the application:

```bash
argocd app delete guestbook --cascade
```

Verify:

```bash
argocd app list
```

---

# Refresh an Application

Force Argo CD to immediately compare the Git repository with the cluster state:

```bash
argocd app get guestbook --refresh
```

This is useful after pushing changes when you do not want to wait for the next reconciliation cycle.

---

# Synchronization States

| Status        | Meaning                                             |
| ------------- | --------------------------------------------------- |
| **Synced**    | The cluster matches the desired state in Git.       |
| **OutOfSync** | Differences exist between Git and the cluster.      |
| **Unknown**   | Argo CD cannot determine the synchronization state. |

---

# Health States

| Health          | Description                                      |
| --------------- | ------------------------------------------------ |
| **Healthy**     | Application is running as expected.              |
| **Progressing** | Resources are still being created or updated.    |
| **Degraded**    | One or more resources are unhealthy.             |
| **Missing**     | Expected resources are not found in the cluster. |
| **Suspended**   | Resource is intentionally paused.                |

---

# Troubleshooting

## Pods Are Not Running

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

## LoadBalancer External IP Is Pending

Possible causes:

* Cloud load balancer provisioning is still in progress.
* Your Kubernetes environment does not support `LoadBalancer` services (e.g., Kind or Minikube).
* Cloud controller manager is not configured.

Possible solutions:

* Wait a few minutes for the cloud provider to provision the load balancer.
* Use `kubectl port-forward` for local development.
* Install MetalLB for bare-metal or local clusters.

---

## Unable to Access the Web UI

Verify the service:

```bash
kubectl get svc -n argocd
```

Ensure:

* The `EXTERNAL-IP` has been assigned.
* Port **443** is open in your firewall or security group.
* The `argocd-server` pod is running.

---

## Authentication Issues

Retrieve the initial password again:

```bash
argocd admin initial-password -n argocd
```

Or retrieve it from the Kubernetes Secret:

```bash
kubectl get secret argocd-initial-admin-secret \
-n argocd \
-o jsonpath="{.data.password}" | base64 -d && echo
```

---

## Git Repository Access Issues

Common causes include:

* Incorrect repository URL
* Invalid SSH key or Personal Access Token (PAT)
* Missing repository permissions
* Network connectivity issues

Verify repository connectivity from the Argo CD UI or inspect the `argocd-repo-server` logs.

---

# Common CLI Commands

## Login

```bash
argocd login <ARGOCD_SERVER> --insecure
```

## List Applications

```bash
argocd app list
```

## Get Application Details

```bash
argocd app get <APP_NAME>
```

## Synchronize an Application

```bash
argocd app sync <APP_NAME>
```

## Refresh an Application

```bash
argocd app get <APP_NAME> --refresh
```

## View Synchronization History

```bash
argocd app history <APP_NAME>
```

## Delete an Application

```bash
argocd app delete <APP_NAME> --cascade
```

---

# Useful Kubernetes Commands

List Argo CD pods:

```bash
kubectl get pods -n argocd
```

List services:

```bash
kubectl get svc -n argocd
```

List deployments:

```bash
kubectl get deployment -n argocd
```

View all resources:

```bash
kubectl get all -n argocd
```

Restart the Argo CD server:

```bash
kubectl rollout restart deployment argocd-server -n argocd
```

Restart all deployments:

```bash
kubectl rollout restart deployment -n argocd
```

---

# Security Best Practices

* Change the default `admin` password after the first login.
* Enable Single Sign-On (SSO) using your identity provider (LDAP, OIDC, GitHub, Google, Microsoft Entra ID).
* Grant users the minimum RBAC permissions required.
* Use HTTPS with trusted TLS certificates in production.
* Avoid exposing the Argo CD API publicly unless necessary.
* Restrict network access using Kubernetes Network Policies or cloud firewall rules.
* Store Git credentials securely using Kubernetes Secrets or an external secret management solution.

---

# Production Best Practices

* Use separate Git repositories or directories for development, staging, and production environments.
* Protect production branches with pull request reviews.
* Enable automatic synchronization only after validating your deployment process.
* Enable `selfHeal` to prevent configuration drift.
* Enable `prune` to remove obsolete resources.
* Use Git webhooks to reduce synchronization latency.
* Monitor Argo CD metrics with Prometheus and visualize them using Grafana.
* Back up Argo CD configuration and Kubernetes manifests regularly.

---

# Backup and Disaster Recovery

Regularly back up:

* Git repositories
* Kubernetes etcd (or managed cluster backups)
* Argo CD ConfigMaps
* Secrets
* Application manifests

Since Git stores the desired state, restoring a cluster is typically a matter of restoring the cluster itself, reinstalling Argo CD, and reconnecting it to the Git repository.

---

# Interview Questions

### What is GitOps?

GitOps is a deployment methodology where Git serves as the single source of truth for infrastructure and application configuration. Changes are applied to the cluster by synchronizing it with the desired state stored in Git.

---

### How does Argo CD detect changes?

Argo CD detects changes by:

* Polling the Git repository approximately every **3 minutes** by default.
* Receiving Git webhooks for near real-time updates.

---

### What is configuration drift?

Configuration drift occurs when the live Kubernetes resources differ from the manifests stored in Git. Argo CD identifies this difference and can automatically restore the desired state if self-healing is enabled.

---

### What is the difference between Sync and Refresh?

* **Refresh** updates Argo CD's view of the application by rechecking the Git repository and cluster state.
* **Sync** applies changes from Git to the Kubernetes cluster.

---

### What is Self-Heal?

Self-heal automatically restores resources if they are modified manually outside of Git.

---

### What is Prune?

Prune removes Kubernetes resources that have been deleted from the Git repository.

---

# Argo CD Cheat Sheet

| Task                | Command                                                      |
| ------------------- | ------------------------------------------------------------ |
| Login               | `argocd login <SERVER> --insecure`                           |
| List applications   | `argocd app list`                                            |
| Create application  | `argocd app create ...`                                      |
| Sync application    | `argocd app sync <APP_NAME>`                                 |
| Refresh application | `argocd app get <APP_NAME> --refresh`                        |
| View history        | `argocd app history <APP_NAME>`                              |
| Delete application  | `argocd app delete <APP_NAME> --cascade`                     |
| Get pods            | `kubectl get pods -n argocd`                                 |
| Get services        | `kubectl get svc -n argocd`                                  |
| Restart server      | `kubectl rollout restart deployment argocd-server -n argocd` |

---

# References

* Argo CD Official Documentation: https://argo-cd.readthedocs.io/
* Argo CD GitHub Repository: https://github.com/argoproj/argo-cd
* CNCF GitOps Working Group: https://github.com/gitops-working-group

---

# Conclusion

You have learned how to:

* Install Argo CD on Kubernetes
* Access the Web UI and CLI
* Create and manage applications
* Understand manual and automatic synchronization
* Configure self-healing and pruning
* Use Git webhooks for near real-time deployments
* Troubleshoot common issues
* Apply production-ready security and operational best practices

Argo CD, combined with GitOps principles, provides a reliable, auditable, and repeatable approach to Kubernetes application delivery.
