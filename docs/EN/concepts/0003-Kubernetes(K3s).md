# K3s

## Overview

K3s is a lightweight Kubernetes distribution designed to simplify installation, operation, and maintenance while remaining compatible with the Kubernetes API. It is suitable for bare-metal machines, Virtual Machines, edge computing, home labs, CI/CD, and development environments.

K3s packages the essential components required for a Kubernetes cluster, reducing external dependencies and the amount of manual configuration required from administrators.

### Why use K3s?

A conventional Kubernetes installation may require administrators to configure multiple components, certificates, networking, and datastores. K3s reduces this complexity by providing essential components and sensible defaults within a single distribution.

For example, K3s can be installed on a single Linux machine to create a functional Kubernetes cluster. Additional machines can then join the cluster as Agent Nodes to provide more workload capacity.

### Key Features

- **Lightweight Distribution:** Simplifies Kubernetes installation and management.
- **Simple Installation:** Provides an installation script that configures K3s as a system service.
- **Single-Node Cluster:** A single Server Node can run a complete Kubernetes cluster.
- **Server and Agent Nodes:** Supports separating control-plane responsibilities from workload execution.
- **Built-in Components:** Includes commonly used components such as containerd, Flannel, CoreDNS, Traefik, and Local Path Provisioner.
- **Flexible Datastore:** Uses SQLite by default for single-server setups and supports embedded etcd and external databases.
- **Kubernetes Compatibility:** Supports the Kubernetes API and standard tools such as `kubectl` and YAML manifests.

K3s is not a separate Kubernetes API. It is a Kubernetes distribution that packages components and sensible defaults to simplify deployment and operation.

## Architecture & Workflow

### Main Components

#### 1. K3s Server

A Server Node runs `k3s server` and manages the main control-plane components and the datastore according to the selected deployment architecture.

Its responsibilities include:

- Providing the Kubernetes API Server for clients and nodes.
- Managing cluster state through the datastore.
- Running the Scheduler and Controllers.
- Managing node registration and related configuration.

A K3s Server also runs a kubelet, container runtime, and networking components, so it can run workloads as well. A Server Node does not necessarily serve exclusively as a control-plane machine.

#### 2. K3s Agent

An Agent Node runs `k3s agent` to join an existing cluster.

Its responsibilities include:

- Registering the node with the cluster.
- Working with the kubelet to manage assigned Pods.
- Running containers through the container runtime.
- Communicating with the Kubernetes API through a Server.

An Agent does not manage the cluster's control plane or datastore.

#### 3. Datastore

The datastore stores the state and information associated with a Kubernetes cluster.

K3s supports several datastore configurations:

- **SQLite:** The default for single-server setups.
- **Embedded etcd:** Used for high-availability (HA) control planes, generally requiring at least three Server Nodes.
- **External Datastore:** Supports external databases such as MySQL, PostgreSQL, or etcd.

The datastore choice affects high availability, backup, and recovery strategies.

#### 4. Built-in Components

K3s includes several essential or commonly used components:

- **containerd:** The default container runtime.
- **Flannel:** A Container Network Interface (CNI) for Pod networking.
- **CoreDNS:** Provides DNS services within the cluster.
- **Traefik:** An Ingress controller installed by default.
- **ServiceLB:** Enables basic support for LoadBalancer Services without requiring an external load-balancer controller in common configurations.
- **Local Path Provisioner:** Creates Persistent Volumes using local storage on a node.

Some components can be disabled or replaced depending on the cluster design.

### How It Works

1. Install K3s on a Linux machine to create a Server Node.
2. K3s starts the control-plane components, datastore, and required supporting components.
3. The user submits Kubernetes manifests through `kubectl` to the Kubernetes API Server.
4. Controllers reconcile the desired state by creating or updating the relevant resources.
5. The Scheduler selects nodes on which Pods should run.
6. The kubelet and container runtime on each node work together to start containers.
7. Additional machines can be configured with K3s and registered as Agent Nodes in the existing cluster.

### Architecture Overview

```text
                 User / kubectl
                       |
                       v
                K3s Server Node
                ┌──────────────┐
                │ API Server   │
                │ Scheduler    │
                │ Controllers  │
                │ Datastore    │
                └──────┬───────┘
                       |
                 Kubernetes API
                       |
             ┌─────────┴─────────┐
             |                   |
             v                   v
       K3s Agent Node       K3s Agent Node
       ┌────────────┐       ┌────────────┐
       │ Kubelet    │       │ Kubelet    │
       │ containerd │       │ containerd │
       │ Pods       │       │ Pods       │
       └────────────┘       └────────────┘
```

This is a conceptual overview. In a single-node setup, the Server components and workloads can run on the same machine.

## Configuration Reference

### 1. Install K3s

On a Linux machine that will serve as the Server Node, install K3s using the installation script:

```bash
curl -sfL https://get.k3s.io | sh -
```

The script installs K3s and configures it as a system service, generally named `k3s`.

Review the installation script before executing it, particularly on production machines. Choose an explicit version when you need to control which version of K3s is installed.

### 2. K3s Configuration File

By default, K3s reads its configuration from:

```text
/etc/rancher/k3s/config.yaml
```

Example Server configuration:

```yaml
node-name: "k3s-server"

# Optional: Add DNS names or IP addresses
# used to access the API Server.
# tls-san:
#   - "k3s.example.local"
```

- `node-name`: Sets the node name.
- `tls-san`: Adds DNS names or IP addresses that should be included in the Server's TLS certificate.

After changing the configuration, the service may need to be restarted:

```bash
sudo systemctl restart k3s
```

Review the effects of each option before changing it, particularly options involving networking, the datastore, or cluster membership.

### 3. Join an Agent Node

On the Server, the node registration token is available at:

```text
/var/lib/rancher/k3s/server/node-token
```

Install K3s on the Agent machine by specifying the Server URL and token:

```bash
curl -sfL https://get.k3s.io | \
  K3S_URL=https://<server-ip>:6443 \
  K3S_TOKEN=<node-token> sh -
```

- `K3S_URL`: The URL of the K3s Server that the Agent connects to.
- `K3S_TOKEN`: The token used to authenticate the node's registration.
- `6443`: The Kubernetes API port used in this example.

Ensure that firewall rules, network connectivity, and node hostnames are configured appropriately. Never expose the node token to unauthorized users.

### 4. kubectl and Kubeconfig

K3s creates an administrative kubeconfig file at:

```text
/etc/rancher/k3s/k3s.yaml
```

Check the cluster nodes:

```bash
sudo k3s kubectl get nodes
```

List Pods across all namespaces:

```bash
sudo k3s kubectl get pods -A
```

This kubeconfig provides high-level cluster access and must be protected accordingly. Accessing the cluster from another machine requires appropriate server addressing, TLS certificates, and user permissions.

### 5. Manage K3s Components

K3s installs several components by default, including Traefik and ServiceLB. Check which components are already running before installing other systems with overlapping responsibilities.

For example, Traefik can be disabled during installation:

```bash
curl -sfL https://get.k3s.io | \
  sh -s - server --disable=traefik
```

Applying similar changes to an existing cluster requires planning for configuration consistency and workload impact. Disabling a component is not equivalent to deleting its Pod with `kubectl delete`.

### 6. Persistence and Storage

K3s includes a Local Path Provisioner by default. It can provision Persistent Volumes using storage available on a node.

However, local storage is tied to the machine holding the data. Backup, recovery, and workload migration must be considered if data needs to survive node failures.

For workloads containing important data, select a storage solution appropriate for the required availability and operational characteristics.

## Common Workflow

### 1. Check Installation

Check the Server service:

```bash
sudo systemctl status k3s
```

Follow the Server logs:

```bash
sudo journalctl -u k3s -f
```

The `-f` option follows new log entries continuously.

### 2. Check Cluster Status

```bash
sudo k3s kubectl get nodes -o wide
sudo k3s kubectl get pods -A
```

Use these commands to inspect node status, IP addresses, and Pods across the cluster.

### 3. Deploy an Application

Apply a Deployment manifest:

```bash
sudo k3s kubectl apply -f deployment.yaml
```

Check the result:

```bash
sudo k3s kubectl rollout status deployment/<deployment-name>
sudo k3s kubectl get pods
```

### 4. Join Additional Nodes

Install K3s on a new machine using `K3S_URL` and `K3S_TOKEN`, then verify the node registration on the Server:

```bash
sudo k3s kubectl get nodes
```

If the node does not appear, inspect service status, Agent logs, the token, server address, and firewall configuration.

### 5. Debug Workloads

Inspect Pod details:

```bash
sudo k3s kubectl describe pod <pod-name>
```

Read Pod logs:

```bash
sudo k3s kubectl logs <pod-name>
```

For Pods with multiple containers, specify a container using `-c <container-name>`.

### 6. Stop or Restart K3s

On the Server:

```bash
sudo systemctl stop k3s
sudo systemctl start k3s
```

On an Agent:

```bash
sudo systemctl stop k3s-agent
sudo systemctl start k3s-agent
```

Stopping a service on a node can affect workloads and cluster communication. Evaluate the impact before performing these operations on a running service.

### 7. Uninstall K3s

On a Server Node:

```bash
sudo /usr/local/bin/k3s-uninstall.sh
```

On an Agent Node:

```bash
sudo /usr/local/bin/k3s-agent-uninstall.sh
```

Uninstallation may delete the local cluster datastore, configuration, and local Persistent Volume data. Back up required data and assess the impact before proceeding.

### Important Notes

- K3s is a Kubernetes distribution, not a container runtime or container image builder.
- Server Nodes can run workloads, so Pods are not necessarily scheduled exclusively on Agents.
- A single-server setup using SQLite does not automatically provide high availability for the control plane.
- Agents need valid network connectivity and authentication to join the Server.
- K3s includes networking, ingress, and storage components by default. Inspect these components before installing alternatives with overlapping responsibilities.
- Kubeconfig files and node tokens are security-sensitive and must be protected.
- Production deployments require appropriate backup, upgrade, high-availability, security, and monitoring strategies.
