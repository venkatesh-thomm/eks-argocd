#### Argo CD Repository Structure (App of Apps Pattern)



**Repository Structure**

```text
eks-argocd/
│
├── applications/
│   ├── namespace.yaml
│   ├── roboshop.yaml
│   └── storage-classes.yaml
│
├── namespaces/
│   ├── flipkart.yaml
│   └── roboshop.yaml
│
├── roboshop/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── pvc.yaml
│   └── ...
│
├── storage-classes/
│   ├── gp2.yaml
│   └── gp3.yaml
│
├── app-of-apps.yaml
│
├── installation.md
├── application.md
└── ArgoCD Hands-on Guide.md
```

---

**Repository Flow**

The deployment starts from a single file called **app-of-apps.yaml**.

```text
                    app-of-apps.yaml
                           │
                           ▼
                 Parent Argo CD Application
                           │
                           ▼
                 applications/ directory
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
 namespace.yaml     roboshop.yaml    storage-classes.yaml
        │                  │                  │
        ▼                  ▼                  ▼
 namespaces/         roboshop/       storage-classes/
        │                  │                  │
        ▼                  ▼                  ▼
 Kubernetes       Application       Storage Classes
 Namespaces        Resources
```

---

**1. app-of-apps.yaml**

This is the **Parent Application**.

It is the only manifest that needs to be applied manually.

```bash
kubectl apply -f app-of-apps.yaml
```

Its responsibility is to tell Argo CD:

- Which Git repository to monitor
- Which directory contains child applications

Example:

```yaml
spec:
  source:
    repoURL: https://github.com/venkatesh-thomm/eks-argocd.git
    path: applications
```

Once this application is created, Argo CD automatically discovers every application defined in the `applications/` directory.

---

**2. applications/**

The `applications/` directory contains **child Argo CD Application resources**.

Each YAML file in this directory creates an independent Argo CD Application.

Repository layout:

```text
applications/

namespace.yaml

roboshop.yaml

storage-classes.yaml
```

Instead of deploying Kubernetes resources directly, these files instruct Argo CD where to find them.

Think of this directory as an **index of applications**.

---

**namespace.yaml**

Purpose:

Deploy all Namespace manifests stored under:

```text
namespaces/
```

Example:

```yaml
spec:
  source:
    path: namespaces
```

This application creates Kubernetes namespaces such as:

- roboshop
- flipkart

---

**roboshop.yaml**

Purpose:

Deploy the Roboshop application.

It points Argo CD to:

```text
roboshop/
```

The directory contains application resources such as:

- Deployments
- Services
- ConfigMaps
- Secrets
- Ingress
- PersistentVolumeClaims

---

**storage-classes.yaml**

Purpose:

Deploy StorageClass resources.

It points Argo CD to:

```text
storage-classes/
```

Typical resources include:

- gp2 StorageClass
- gp3 StorageClass

---

**3. namespaces/**

This directory contains standard Kubernetes Namespace manifests.

Example:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: flipkart
```

After synchronization, Kubernetes creates:

```text
Namespace
---------
flipkart
```

Similarly,

```text
roboshop.yaml
```

creates

```text
Namespace
---------
roboshop
```

Nothing else is deployed from this directory.

---

**4. roboshop/**

This directory contains the actual application manifests.

Example:

```text
roboshop/

frontend.yaml

catalogue.yaml

user.yaml

cart.yaml

shipping.yaml

payment.yaml

redis.yaml

mysql.yaml

mongodb.yaml
```

These manifests create Kubernetes resources such as:

- Deployment
- Service
- ConfigMap
- Secret
- PVC
- Ingress

---

**5. storage-classes/**

Contains Kubernetes StorageClass resources.

Example:

```yaml
kind: StorageClass
metadata:
  name: gp3
```

These StorageClasses are later used by PersistentVolumeClaims (PVCs).

---

**Why Separate These Directories?**

Separating infrastructure from applications improves readability and maintainability.

**Infrastructure**

```text
Namespaces

Storage Classes

RBAC

Network Policies
```

Infrastructure changes infrequently.

---

**Applications**

```text
Deployments

Services

Ingress

Secrets

ConfigMaps
```

Applications change frequently.

Keeping them separate makes management easier.

---

**Deployment Order**

The deployment follows this sequence:

```text
Step 1

kubectl apply app-of-apps.yaml

        │

        ▼

Step 2

Parent Application

        │

        ▼

Step 3

Reads applications/

        │

        ▼

Creates Child Applications

        │

        ├───────────────┐
        │               │
        ▼               ▼
Namespaces        Storage Classes
        │               │
        └───────┬───────┘
                ▼
         Roboshop Application
                │
                ▼
      Deployments
      Services
      Ingress
      ConfigMaps
      PVCs
```

---

**Why Create Namespaces First?**

Applications are deployed into namespaces.

If the namespace does not exist, deployments fail.

Correct sequence:

```text
Namespace
      │
      ▼
Deployment
      │
      ▼
Service
      │
      ▼
Ingress
```

This is why namespace creation is separated into its own application.

---

**Why Create Storage Classes First?**

PersistentVolumeClaims depend on StorageClasses.

Correct sequence:

```text
StorageClass
      │
      ▼
PersistentVolumeClaim
      │
      ▼
Pod
```

If the StorageClass is missing, PVCs remain in the **Pending** state.

---

**Advantages of the App of Apps Pattern**

- Single entry point (`app-of-apps.yaml`)
- Easy to manage multiple applications
- Modular repository structure
- Independent synchronization of applications
- Better scalability
- Clear separation of infrastructure and application resources
- Simplifies onboarding for new team members

---

**Summary**

| Directory | Purpose |
|----------|---------|
| `app-of-apps.yaml` | Parent Argo CD Application |
| `applications/` | Child Argo CD Applications |
| `namespaces/` | Kubernetes Namespace manifests |
| `roboshop/` | Application manifests (Deployments, Services, etc.) |
| `storage-classes/` | StorageClass manifests |
| `installation.md` | Installation documentation |
| `application.md` | Application deployment guide |
| `ArgoCD Hands-on Guide.md` | Complete Argo CD learning guide |

---

**Overall Workflow**

```text
Developer
      │
      ▼
Push Code to GitHub
      │
      ▼
Argo CD Parent Application
      │
      ▼
Reads applications/
      │
      ▼
Creates Child Applications
      │
      ▼
Deploy Infrastructure
      │
      ▼
Deploy Applications
      │
      ▼
Kubernetes Cluster
```
