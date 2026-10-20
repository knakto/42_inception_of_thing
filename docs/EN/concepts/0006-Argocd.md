# Argo CD

## Overview

Argo CD is a declarative GitOps Continuous Delivery (CD) tool for Kubernetes. It uses a Git repository as the primary source of truth for an application's desired state and Kubernetes resources.

Argo CD compares the configuration stored in Git with the live state of the Kubernetes cluster. It reports differences between the two states and can synchronize resources manually or automatically according to the configured synchronization policy.

### Why use Argo CD?

In a conventional deployment workflow, a CI/CD pipeline may need credentials to access a Kubernetes cluster and deploy resources directly. This requires managing deployment permissions and Kubernetes operations within the pipeline.

Argo CD separates these responsibilities. A CI system handles testing, building, and pushing container images, while Argo CD applies configuration from Git to the Kubernetes cluster.

For example, when an image tag in a manifest is updated and committed to Git, Argo CD can detect the change and deploy the new version according to its synchronization policy.

### Key Features

- **GitOps:** Uses Git repositories as the source of truth for application desired state.
- **Continuous Reconciliation:** Compares Git configuration with the live state in the cluster.
- **Automated Synchronization:** Synchronizes configuration automatically according to defined policies.
- **Drift Detection:** Detects resource changes that differ from the Git configuration.
- **Application Health Monitoring:** Reports synchronization status and application health.
- **Multi-Cluster Management:** Manages deployments across multiple Kubernetes clusters.
- **Configuration Management Integration:** Supports plain YAML, Kustomize, Helm, and other configured configuration-management tools.
- **Access Control:** Defines application permissions and boundaries through RBAC and AppProjects.

Argo CD does not automatically build container images or run automated tests. These responsibilities are typically handled by CI tools such as GitHub Actions, GitLab CI, or Jenkins.

## Architecture & Workflow

### Main Components

#### 1. Argo CD API Server

The API Server provides an interface for the Web UI, Argo CD CLI, and external systems that manage applications.

Its responsibilities include:

- Managing applications and reporting status.
- Accepting synchronization requests and related operations.
- Handling authentication and RBAC.
- Managing repository and cluster connection configuration according to the required permissions.
- Receiving webhook events that can accelerate change detection.

#### 2. Repository Server

The Repository Server reads configuration from Git repositories or supported sources and generates Kubernetes manifests from the configured files.

Its responsibilities include:

- Retrieving repository contents and the requested revision.
- Reading files from the path specified in an Application.
- Rendering Helm charts or processing configurations through Kustomize and supported tools.
- Returning rendered manifests for comparison with the live state.

The Repository Server uses caching to reduce repeated repository access.

#### 3. Application Controller

The Application Controller is a Kubernetes controller that monitors application state and compares the desired state with the live state.

Its responsibilities include:

- Detecting differences between Git and the cluster.
- Reporting `Synced` or `OutOfSync` status.
- Evaluating application health.
- Performing synchronization when requested or permitted by the automated sync policy.
- Continuously reconciling application state.

#### 4. Application

An Application is an Argo CD Custom Resource that specifies where application configuration is stored and which Kubernetes cluster should receive the deployment.

Its main elements include:

- **Source:** The repository, revision, and path or chart used to generate resources.
- **Destination:** The target cluster and namespace.
- **Project:** The AppProject that controls the application's boundaries.
- **Sync Policy:** The policy for synchronization and resource management.

An Application is not the running application itself. It is a resource Argo CD uses to declare and track application deployments.

#### 5. AppProject

An AppProject groups applications and defines the boundaries within which each application may operate.

An AppProject can control:

- Which Git repositories are permitted.
- Which clusters and namespaces applications may target.
- Which Kubernetes resource types may be created.
- Which roles and permissions are available for managing applications.

Every Application belongs to a project. The `default` project is available automatically for initial use, but it has broad permissions. For real deployments, create AppProjects with explicit restrictions.

#### 6. Supporting Components

Argo CD includes other components that support its operation:

- **Redis:** Caches data to improve performance.
- **Dex:** An identity service that may provide Single Sign-On in configurations that use it.
- **ApplicationSet Controller:** Generates and manages many Applications from templates and configured data sources.
- **Notifications:** Sends notifications about application events and status when enabled and configured.

Some components are optional or depend on the installation model.

### How It Works

1. A developer pushes code to a Git repository, while the CI pipeline performs testing, builds the application, and pushes the container image according to the team's workflow.
2. Kubernetes configuration, such as an image tag or Helm values, is committed to the repository tracked by Argo CD.
3. Argo CD reads the configuration from Git and generates the desired Kubernetes manifests.
4. The Application Controller compares the desired state with the live state in the cluster.
5. If differences are detected, the application is marked `OutOfSync`.
6. The user can synchronize the application manually or configure automatic synchronization.
7. Kubernetes controllers manage the workloads represented by the applied resources.
8. Argo CD continues monitoring the application and reporting synchronization status and health.

### Architecture Overview

```text
       Developer
           |
           v
       Git Repository <--------- CI Pipeline
           |                     Build / Test /
           |                     Push Image
           v
       Argo CD
       ├── API Server
       ├── Repository Server
       └── Application Controller
                   |
            Compare Desired State
              with Live State
                   |
                   v
            Kubernetes API Server
                   |
                   v
          Kubernetes Resources
          ├── Deployment
          ├── Service
          └── ConfigMap
```

Git defines the desired state, while the Application Controller reconciles the live cluster state according to the configured policy. A change in Git is not necessarily deployed immediately if synchronization has not been requested or automated sync is disabled.

### Important Status Concepts

- **Synced:** The resources match the desired state calculated by Argo CD from the configured source.
- **OutOfSync:** The live state differs from the desired state.
- **Healthy:** The application is assessed as healthy by Argo CD.
- **Progressing:** Some resources are still changing state or have not finished progressing.
- **Degraded:** Resources or the application have an unhealthy status according to Argo CD's health assessment.

Synchronization status and health are separate concepts. An application can be Synced but not Healthy, such as when a Deployment matches Git but its Pods are not ready to serve requests.

## Configuration Reference

### 1. Install Argo CD

This example installs Argo CD on an existing Kubernetes cluster, such as a cluster created with k3d.

Create the namespace:

```bash
kubectl create namespace argocd
```

Install the components using the official installation manifest:

```bash
kubectl apply --server-side --force-conflicts \
  -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Check the status:

```bash
kubectl get pods -n argocd
```

This example uses the stable installation manifest for initial experimentation. For production, select an explicit Argo CD version and follow the installation and upgrade guidance for that version.

### 2. Access the Web UI

For local development, forward a port from the host to the Argo CD Server:

```bash
kubectl port-forward svc/argocd-server \
  -n argocd 8080:443
```

Then open:

```text
https://localhost:8080
```

This method does not require exposing Argo CD directly to an external network.

A standard installation may create an initial admin password in a Secret. Retrieve it using:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
echo
```

The initial username is generally `admin`. Change the password and protect the credentials after initial access according to the guidance for the installed version.

### 3. Application Manifest

Applications can be declared through Kubernetes YAML like other resources.

Example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: guestbook
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```

Field descriptions:

- `apiVersion`: The API version of the Argo CD Application resource.
- `kind`: Identifies the resource as an `Application`.
- `metadata.name`: The Application name in Argo CD.
- `metadata.namespace`: The namespace containing the Application resource, generally `argocd`.
- `spec.project`: The AppProject used by the Application.
- `spec.source.repoURL`: The Git repository containing the application configuration.
- `spec.source.targetRevision`: The branch, tag, or commit to track.
- `spec.source.path`: The directory containing the application configuration.
- `spec.destination.server`: The target Kubernetes API Server.
- `spec.destination.namespace`: The target deployment namespace.
- `CreateNamespace=true`: Allows Argo CD to create the destination namespace if it does not exist.

This example uses a public repository for experimentation and does not enable automated synchronization. You can inspect the changes and synchronize manually first.

Create the Application:

```bash
kubectl apply -f application.yaml
```

### 4. Sync Policy

Applications generally use manual synchronization unless automated synchronization is configured.

To enable automated synchronization when Argo CD detects changes, add configuration such as:

```yaml
spec:
  syncPolicy:
    automated:
      enabled: true
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Important settings:

- `enabled: true`: Enables automated synchronization.
- `prune: true`: Allows Argo CD to delete managed resources that are no longer present in the desired state.
- `selfHeal: true`: Allows Argo CD to correct drift caused by changes made directly in the cluster rather than through Git.
- `CreateNamespace=true`: Creates the destination namespace if it does not exist.

**Warning:** `prune: true` can delete resources removed from Git, while `selfHeal: true` can overwrite manual changes made in the cluster. Start with manual synchronization while learning, then enable automation after understanding its effects.

### 5. Helm Integration

Argo CD can use Helm charts as an Application source.

Example Application using a chart from a Helm repository:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://prometheus-community.github.io/helm-charts
    chart: prometheus
    targetRevision: "<chart-version>"
    helm:
      releaseName: prometheus
      values: |
        alertmanager:
          enabled: false
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```

`<chart-version>` is a placeholder that must be replaced with an existing chart version before use.

When Helm is used through Argo CD, Helm renders the chart into Kubernetes manifests, and Argo CD manages the application lifecycle through its own reconciliation and release-management system. This differs from independently installing a chart using `helm install`.

For larger configurations, values can be stored in Git or supplied through supported external values-file references.

### 6. Multi-Environment Configuration

Multiple environments can reuse the same chart or manifest templates while maintaining separate configuration values.

Example repository structure:

```text
gitops/
├── apps/
│   ├── dev/
│   │   └── values.yaml
│   ├── staging/
│   │   └── values.yaml
│   └── production/
│       └── values.yaml
└── charts/
    └── myapp/
        ├── Chart.yaml
        ├── values.yaml
        └── templates/
```

Environment separation can use directories, Helm values, Kustomize overlays, or separate repositories, depending on the requirements.

Apply appropriate permissions and destination cluster or namespace restrictions for each environment, especially Production.

## Common Workflow

### 1. Check Argo CD

Check the Pods:

```bash
kubectl get pods -n argocd
```

Check the Application resources:

```bash
kubectl get applications -n argocd
```

### 2. Create an Application

Apply an Application manifest containing the source and destination configuration:

```bash
kubectl apply -f application.yaml
```

Verify that the Application exists:

```bash
kubectl get applications -n argocd
```

### 3. Inspect Application Status

After installing and logging in to the Argo CD CLI, inspect the applications:

```bash
argocd app list
argocd app get guestbook
```

Compare the desired state with the live state:

```bash
argocd app diff guestbook
```

These commands help identify whether the application is Synced or OutOfSync and which resources require changes.

### 4. Sync an Application

Synchronize manually:

```bash
argocd app sync guestbook
```

Wait for the sync operation to complete:

```bash
argocd app wait guestbook
```

If automated sync is enabled and applicable, synchronization may occur automatically when Argo CD detects a change.

### 5. Update an Application

Typical update workflow:

1. Build and test the application through CI.
2. Push the image to a container registry.
3. Update the image tag or configuration in the Git repository tracked by Argo CD.
4. Commit and push the change.
5. Argo CD detects the new configuration.
6. Synchronization occurs according to the configured policy.
7. Check synchronization status and application health.

Pushing source code or a container image does not automatically cause Argo CD to deploy a new version. If the tracked configuration has not changed, a new deployment may not occur.

### 6. Roll Back an Application

In a GitOps workflow, it is generally preferable to restore the desired configuration in Git to a previous configuration or commit and then synchronize it.

For example:

```bash
git revert <commit-sha>
git push
```

This keeps Git as the source of truth and preserves an auditable history of configuration changes.

Rolling back directly through Argo CD may be unsuitable for some automated-sync configurations because the cluster can be synchronized back to the state defined in Git.

### 7. Delete an Application

Delete the Application resource:

```bash
kubectl delete -f application.yaml
```

Deleting an Application may affect the Kubernetes resources it manages, depending on its finalizers and deletion policy. Assess the impact before proceeding, particularly in production.

### Important Notes

- Git represents the desired state, while the Kubernetes cluster holds the live state.
- `OutOfSync` does not necessarily mean an application is down; it means the live state differs from the desired state.
- Automated sync, pruning, and self-healing are distinct behaviors with different consequences.
- Helm used through Argo CD has a different lifecycle-management model from a chart installed independently through Helm CLI.
- Users who can write to a Git repository trusted by Argo CD may influence the resources deployed to the cluster. Repository permissions must be controlled carefully.
- Restrict AppProjects, RBAC, and destination clusters to the permissions applications actually need.
- GitOps improves change traceability but does not replace Secrets Management, Testing, Monitoring, or Backup planning.
