# k3d

## Overview

k3d is a tool for creating and managing K3s clusters inside Docker. Each Kubernetes Node runs inside a Docker Container instead of requiring a separate physical machine or Virtual Machine.

k3d is suitable for local development, testing, CI/CD, and Kubernetes experimentation because it can quickly create multi-node clusters on a single machine and remove them when they are no longer needed.

### Why use k3d?

Experimenting with Kubernetes may require multiple nodes to test workload scheduling, networking, and interactions between components. Creating a separate VM for every node can introduce additional resource requirements and setup complexity.

k3d simplifies this process by using Docker Containers instead of separate machines, making it easier to develop and test Kubernetes environments locally.

### Key Features

- **Lightweight Clusters:** Uses Docker Containers as Kubernetes Nodes.
- **Multi-Node Support:** Configures the number of Server and Agent Nodes.
- **Fast Lifecycle Management:** Creates, starts, stops, and deletes clusters through the CLI.
- **Port Mapping:** Exposes services through Docker port mappings and a Load Balancer.
- **Local Image Import:** Imports locally built container images into a cluster.
- **Declarative Configuration:** Defines cluster creation settings in a YAML configuration file.
- **Multiple Clusters:** Creates separate clusters for different development and testing environments.

### Relationship Between Kubernetes, K3s, and k3d

- **Kubernetes:** A system for managing containerized workloads.
- **K3s:** A lightweight Kubernetes distribution designed for simplified installation and operation.
- **k3d:** A tool for creating and managing K3s clusters inside Docker.
- **Docker:** The container platform used to run the node containers created by k3d.

k3d is not a Kubernetes distribution and does not replace K3s. It makes creating and managing K3s clusters more convenient.

## Architecture & Workflow

### Main Components

#### 1. Host Machine

The Host Machine is the physical machine or VM running Docker, the k3d CLI, and tools such as `kubectl`.

Users execute commands such as `k3d cluster create` on the host to request cluster creation.

#### 2. Docker Engine

Docker Engine creates and manages the containers that act as Kubernetes Nodes.

k3d depends on a working Docker Engine, so it cannot create a cluster if Docker is unavailable.

#### 3. K3s Node Containers

k3d creates containers that run K3s. These nodes generally have two roles:

- **Server Node:** Runs the control-plane components and may also run application workloads.
- **Agent Node:** Runs application workloads managed by the cluster.

Although each node is a Docker Container, it still has K3s components such as kubelet and containerd for managing Kubernetes workloads.

#### 4. Load Balancer Container

By default, k3d creates a Load Balancer Container in front of the Server Nodes to help route incoming connections.

- Provides an access point for the Kubernetes API.
- Supports host-to-cluster port forwarding when port mappings are configured.
- Helps avoid binding access to a single Server Node directly.

This load balancer is part of the infrastructure managed by k3d. It can be disabled through supported configuration options.

#### 5. Docker Network

k3d creates a Docker Network for each cluster to allow node containers to communicate.

However, the Docker network used by nodes and the Kubernetes Pod network operate at different network layers. Accessing applications from the host may require additional port mappings, Services, or Ingress configuration.

### How It Works

1. The user runs `k3d cluster create` and specifies the cluster name and desired number of nodes.
2. k3d communicates with Docker Engine to create node containers, a Docker network, and a load balancer according to the configuration.
3. K3s starts inside the node containers and initializes the Kubernetes cluster.
4. k3d prepares a kubeconfig so that `kubectl` can access the cluster.
5. The user creates Kubernetes resources through YAML manifests as usual.
6. Kubernetes Schedulers and Controllers manage workloads within the cluster.
7. When testing is complete, the user can stop or delete the cluster through k3d.

### Architecture Overview

```text
                  Host Machine
        ┌────────────────────────────┐
        │ k3d CLI       kubectl      │
        │              Docker Engine │
        └────────────────┬───────────┘
                         |
                   Docker Network
                         |
             ┌───────────┴───────────┐
             |                       |
             v                       v
      Load Balancer           K3s Server Node
      Container               Container
             |                ┌─────────────┐
             └───────────────>│ Control     │
                              │ Plane       │
                              │ kubelet     │
                              │ containerd  │
                              └──────┬──────┘
                                     |
                          ┌──────────┴──────────┐
                          |                     |
                          v                     v
                    Agent Node 0          Agent Node 1
                    Container             Container
                    ┌───────────┐         ┌───────────┐
                    │ kubelet   │         │ kubelet   │
                    │ containerd│         │ containerd│
                    │ Pods      │         │ Pods      │
                    └───────────┘         └───────────┘
```

This diagram illustrates a cluster with one Server Node and two Agent Nodes. Actual layouts may vary depending on the number of nodes, load-balancer configuration, and other options.

## Configuration Reference

### 1. Requirements

A machine running k3d requires:

- A working Docker Engine.
- The k3d CLI.
- `kubectl` to interact with the Kubernetes API.

Verify the installed tools:

```bash
docker version
k3d version
kubectl version --client
```

### 2. Create a Cluster

Create a cluster named `dev` with one Server Node and two Agent Nodes:

```bash
k3d cluster create dev \
  --servers 1 \
  --agents 2
```

- `dev`: The cluster name.
- `--servers 1`: The number of Server Nodes.
- `--agents 2`: The number of Agent Nodes.

This creates a Kubernetes cluster for local development inside Docker. k3d also manages the associated network and load balancer according to its default configuration.

### 3. Port Mapping

To access applications through a host port, define a port mapping when creating the cluster:

```bash
k3d cluster create dev \
  --servers 1 \
  --agents 2 \
  -p "8081:80@loadbalancer"
```

The mapping `"8081:80@loadbalancer"` means:

- `8081`: The port on the host.
- `80`: The port on the load-balancer container.
- `@loadbalancer`: Applies the mapping to the load balancer managed by k3d.

Once an Ingress Controller or suitable service routing configuration is in place, applications can be accessed through the configured host port.

Port mapping alone does not create an application, Kubernetes Service, or Ingress.

### 4. Cluster Configuration File

To keep cluster configuration in Git and recreate the environment consistently, use a k3d YAML configuration file.

Generate a starter configuration:

```bash
k3d config init --output k3d.yaml
```

Edit the generated file to match the desired environment.

One supported configuration format is:

```yaml
apiVersion: k3d.io/v1alpha5
kind: Simple
metadata:
  name: dev

servers: 1
agents: 2

ports:
  - port: "8081:80"
    nodeFilters:
      - loadbalancer
```

Create the cluster from the file:

```bash
k3d cluster create --config k3d.yaml
```

The `apiVersion` of k3d configuration files may change between releases. Prefer the configuration generated by `k3d config init` for the installed version, or consult the official schema documentation.

### 5. Kubeconfig

k3d can manage kubeconfig settings so that `kubectl` can connect to the created cluster.

By default, creating a cluster generally updates the default kubeconfig and switches the current context to the new cluster.

Check the current context:

```bash
kubectl config current-context
```

Check the nodes:

```bash
kubectl get nodes
```

Retrieve a cluster's kubeconfig:

```bash
k3d kubeconfig get dev
```

Kubeconfig files contain connection and authentication information, so they should be protected appropriately.

### 6. Import Local Container Images

After building an image on the host, make it available to the container runtime inside the k3d nodes.

Example:

```bash
docker build -t demo:v1 .
k3d image import demo:v1 --cluster dev
```

The image can then be referenced from a Kubernetes manifest:

```yaml
containers:
  - name: demo
    image: demo:v1
    imagePullPolicy: IfNotPresent
```

Image importing is useful for local development because it avoids pushing every build to a remote registry. The image must be imported into the cluster being used, and the manifest must reference the correct image tag.

### 7. Volumes and Persistent Data

k3d can mount a host directory into the node containers:

```bash
k3d cluster create dev \
  --volume "$PWD/data:/mnt/data@all"
```

This example maps the host's `data` directory to `/mnt/data` inside the node containers matched by the `@all` node filter.

Mounting a host directory into a node does not automatically create a Kubernetes PersistentVolume or PersistentVolumeClaim. Kubernetes storage resources must be configured separately.

Data held inside containers or node-local storage may also be lost when a cluster is deleted unless suitable persistent storage has been configured.

## Common Workflow

### 1. Create the Cluster

```bash
k3d cluster create dev --servers 1 --agents 2
```

Creates a cluster for local development.

### 2. Verify the Cluster

```bash
k3d cluster list
kubectl get nodes -o wide
kubectl get pods -A
```

Verify that the cluster was created successfully and that its nodes are ready.

### 3. Deploy an Application

Apply Kubernetes manifests:

```bash
kubectl apply -f k8s/
```

Inspect the resources:

```bash
kubectl get deployments
kubectl get pods
kubectl get services
```

### 4. Access the Application

For local testing without additional host port mappings, use:

```bash
kubectl port-forward service/<service-name> 8080:80
```

Open `http://localhost:8080` while the command is running.

For Ingress testing or access through a host port, configure port mappings when creating the cluster and set up the required Ingress or Service resources.

### 5. Load a Local Image

```bash
docker build -t demo:v1 .
k3d image import demo:v1 --cluster dev
kubectl apply -f deployment.yaml
```

This workflow is useful when developing an application and testing images built locally.

### 6. Stop and Start the Cluster

Stop the cluster:

```bash
k3d cluster stop dev
```

Start it again:

```bash
k3d cluster start dev
```

Stopping a cluster stops its node containers but does not delete the cluster.

### 7. Delete the Cluster

```bash
k3d cluster delete dev
```

Deletes the cluster and resources managed by k3d for that cluster. Back up important information beforehand because data inside containers or certain volumes may become unrecoverable after deletion.

### Important Notes

- k3d uses Docker to run K3s Nodes rather than simulating each node as a Virtual Machine.
- k3d is suitable for local development and testing, but it does not perfectly reproduce physical machines or production networking.
- Exposing a service may involve Docker port mappings, Kubernetes Services, and Ingress configuration.
- Local container images must be imported into the cluster or published to a registry that the cluster can access.
- Mounting a host directory into a node is not the same as creating Kubernetes persistent storage.
- Deleting a cluster may cause data loss. Back up important data before performing destructive operations.
