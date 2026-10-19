# Kubernetes (K8s)

## Overview

Kubernetes (K8s) is an open-source platform for managing containerized applications. It helps control container workloads within a cluster, including application deployment, replica scaling, workload scheduling across nodes, and application state management.

### Why use Kubernetes?

As an application grows to include multiple containers, managing each container manually becomes more complex. Kubernetes allows users to define the desired state of a system and uses controllers to continuously reconcile the actual state toward that desired state.

For example, if an application should run three replicas, you can define the desired replica count in a configuration file and let Kubernetes manage the Pods accordingly.

### Key Features

- **Container Orchestration:** Manages containerized workloads within a cluster.
- **Self-Healing:** Replaces or recreates Pods managed by controllers when they are missing or fail to match the desired state.
- **Scaling:** Increases or decreases the number of application replicas.
- **Service Discovery & Networking:** Enables workloads to communicate through Services and cluster networking.
- **Declarative Configuration:** Defines desired resources through YAML or JSON manifests.
- **Rolling Updates:** Updates workloads gradually according to the configured Deployment strategy.

Kubernetes does not automatically manage everything. Host machine configuration, application image builds, and backup strategies still require additional tools and configuration appropriate to the system.

## Architecture & Workflow

### Main Components

#### 1. Cluster

A Cluster is a group of machines that work together to run and manage workloads. It consists of a Control Plane and one or more Worker Nodes for running Pods.

#### 2. Control Plane

The Control Plane manages and coordinates the state of the cluster.

- **kube-apiserver:** The entry point for requests through the Kubernetes API.
- **etcd:** A distributed key-value store that holds cluster state and configuration data.
- **kube-scheduler:** Selects suitable nodes for Pods that have not yet been scheduled.
- **kube-controller-manager:** Runs controllers that monitor actual state and attempt to reconcile it with the desired state.

#### 3. Worker Node

Worker Nodes run application workloads within the cluster.

- **kubelet:** An agent on each node that ensures assigned Pods run as specified.
- **Container Runtime:** Software that runs containers, such as containerd or CRI-O.
- **Network Components:** Components that enable Pod and Service networking. The exact implementation depends on the cluster setup and networking plugins.

#### 4. Core Kubernetes Objects

- **Pod:** The basic workload unit managed by Kubernetes. A Pod can contain one or more containers that work together.
- **Deployment:** Declares and manages stateless application workloads, including their ReplicaSets and Pod updates.
- **ReplicaSet:** Maintains the desired number of Pods. ReplicaSets are generally managed by Deployments.
- **Service:** Provides a stable network endpoint for accessing Pods, even when those Pods are recreated or moved.
- **ConfigMap:** Stores non-confidential configuration data.
- **Secret:** Stores sensitive information such as credentials or tokens. Additional security configuration is still required.

### How It Works

1. The user writes YAML manifests to define the desired state of Kubernetes resources.
2. `kubectl apply` sends the configuration to the Kubernetes API Server.
3. The API Server validates requests and persists the relevant state in etcd.
4. Controllers detect resources that need to be created or updated and work toward the desired state.
5. The Scheduler selects nodes for Pods that have not been scheduled.
6. The kubelet on each selected node works with the container runtime to pull images when necessary and start containers.
7. Resource status is updated through the Kubernetes API, allowing the system to monitor and continuously reconcile its state.

### Architecture Overview

```text
             User / kubectl
                    |
                    v
              kube-apiserver
                    |
                    v
                  etcd
                    ^
                    |
         +----------+----------+
         |                     |
         v                     v
     Scheduler          Controllers
         |                     |
         +----------+----------+
                    |
             Assign Pod to Node
                    |
          +---------+---------+
          |                   |
          v                   v
      Worker Node         Worker Node
          |                   |
        kubelet             kubelet
          |                   |
    Container Runtime   Container Runtime
          |                   |
         Pods                 Pods
```

This diagram is a conceptual overview of the major components, not a complete time-ordered representation of every request. Controllers and other components cooperate continuously.

## Configuration Reference

### 1. Kubernetes Manifest

A manifest is a YAML or JSON file used to declare Kubernetes object configuration.

Example:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:stable
          ports:
            - containerPort: 80
```

Important fields:

- `apiVersion`: Specifies the Kubernetes API version used by the resource.
- `kind`: Specifies the resource type.
- `metadata`: Contains identifying information such as the resource name and namespace.
- `spec`: Defines the desired state of the resource.
- `status`: Reports the current state, primarily maintained by the Kubernetes system.

### 2. Deployment

A Deployment manages application Pods and workload updates.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:stable
          ports:
            - containerPort: 80
```

- `replicas`: The desired number of Pod replicas.
- `selector`: Identifies the Pods managed through the Deployment's ReplicaSet.
- `template`: The template used to create Pods.
- `containers`: The list of containers within a Pod.
- `image`: The container image to run.
- `containerPort`: Documents a port used by the container; it does not automatically expose that port externally.

**Important:** The labels in `spec.template.metadata.labels` must match the Deployment selector.

### 3. Service

A Service provides an endpoint for accessing Pods using labels and selectors.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

- `selector`: Selects Pods with matching labels.
- `port`: The port exposed by the Service.
- `targetPort`: The destination port on the application Pod.
- `type: ClusterIP`: Exposes the Service primarily within the cluster.

Other Service types include `NodePort` and `LoadBalancer`. External access may require additional Ingress, Gateway, or networking configuration.

### 4. Apply Configuration

Assume the Deployment and Service manifests are stored in a directory named `k8s/`.

```bash
kubectl apply -f k8s/
```

This command creates or updates resources according to the supplied manifests. A working cluster and correctly configured `kubectl` context are required.

### 5. Scale and Update

Change the number of replicas:

```bash
kubectl scale deployment/nginx --replicas=3
```

Update the Deployment image:

```bash
kubectl set image deployment/nginx nginx=nginx:1.28
```

The image shown here illustrates the command syntax. Select an existing image tag appropriate for your environment before using it.

Check the rollout status:

```bash
kubectl rollout status deployment/nginx
```

Changes made through these commands may not be reflected in the original YAML files. Update the manifests to preserve the intended desired state in source control.

### 6. Namespace

A Namespace groups resources within a cluster to help organize and separate workloads.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: development
```

To create a resource inside a Namespace, specify it in `metadata`:

```yaml
metadata:
  name: nginx
  namespace: development
```

A Namespace is not a complete security boundary by itself. RBAC, Network Policies, and other controls may be required.

## Common Workflow

### 1. Check the Environment

Check the Kubernetes CLI and the connected cluster:

```bash
kubectl version
kubectl config current-context
kubectl get nodes
```

`kubectl` is a CLI for communicating with a cluster; it does not create the cluster itself. Cluster creation requires an appropriate tool or service, such as Minikube, kind, or a managed Kubernetes service.

### 2. Deploy an Application

```bash
kubectl apply -f k8s/
```

Creates or updates resources from the manifests in the cluster.

### 3. Inspect Resources

```bash
kubectl get pods
kubectl get deployments
kubectl get services
kubectl get all
```

These commands display different resource types. `kubectl get all` does not list every resource type in the cluster.

### 4. Debug an Application

Inspect a Pod:

```bash
kubectl describe pod <pod-name>
```

View container logs:

```bash
kubectl logs <pod-name>
```

Follow logs continuously:

```bash
kubectl logs -f <pod-name>
```

For a Pod with multiple containers, specify the container using `-c <container-name>` when necessary.

### 5. Access an Application

For local testing, forward a port from a Service to the host:

```bash
kubectl port-forward service/nginx 8080:80
```

Open `http://localhost:8080` while the port-forward command is running.

### 6. Update and Roll Back

Check deployment progress:

```bash
kubectl rollout status deployment/nginx
```

View revision history:

```bash
kubectl rollout history deployment/nginx
```

Roll back to a previous revision:

```bash
kubectl rollout undo deployment/nginx
```

Rollback uses the available Deployment revision history. It does not automatically revert database data or application-managed persistent data.

### 7. Delete Resources

Delete resources created from manifests:

```bash
kubectl delete -f k8s/
```

Or delete a Deployment directly:

```bash
kubectl delete deployment nginx
```

Always verify the target resources before deletion, especially in shared or production clusters.

### Important Notes

- Kubernetes manages the desired state of workloads rather than simply starting containers once.
- Pods may be recreated, so their names and IP addresses should not be treated as permanent endpoints.
- Deployments are suitable for stateless workloads. Stateful workloads may require StatefulSets and Persistent Volumes.
- `kubectl apply` manages resources through manifests, but YAML files alone do not guarantee identical environments across different systems.
- Cluster capabilities depend on the provider, container runtime, networking plugins, storage configuration, and software versions.
- Separate Development, Testing, and Production configurations appropriately, and protect Secrets and access permissions.
