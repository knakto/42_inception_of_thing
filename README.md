# Kubernetes Infrastructure Lab

A hands-on infrastructure repository exploring lightweight Kubernetes environments, application networking, and GitOps-style deployment. The work is organized into three implementation tracks, moving from cluster provisioning to application routing and Git-based deployment workflows.

## What This Repository Covers

### P1 — Provisioning a K3s Cluster

Uses Vagrant to create two virtual machines and configure a K3s Server/Agent setup. The implementation covers VM networking, node identity, token handoff, and `kubectl` access.

### P2 — Application Deployment and Ingress Routing

Deploys three sample web applications to a K3s environment. Kubernetes Deployments, Services, ConfigMaps, shared volumes, and Ingress rules work together to serve application content and route HTTP requests based on the request host.

### P3 — K3d and Argo CD

Creates a local K3d cluster and defines an application deployment sourced from a Git repository through Argo CD. The configuration also explores application versions and the path from host ports through the K3d load balancer and Kubernetes Service to the application container.

These descriptions summarize the repository's intended scope. See the implementation notes for source-level details, known caveats, and suggested verification steps; configuration files alone do not establish that every runtime workflow has been tested successfully.

## Repository Layout

| Path | Purpose |
|---|---|
| [`p1/`](p1/) | Vagrant configuration and scripts for the K3s Server/Agent environment. |
| [`p2/`](p2/) | K3s setup and Kubernetes manifests for the sample applications and Ingress routing. |
| [`p3/`](p3/) | K3d and Argo CD setup, application manifest, and related commands. |
| [`docs/TH/`](docs/TH/) | Thai documentation. |
| [`docs/EN/`](docs/EN/) | English documentation. |
| [`lab/`](lab/) | Separate experimental environment; it is not covered by the P1–P3 implementation-note series. |

## Documentation

Documentation is available in Thai and English. Choose a language and document type below.

### Requirements Checklists

- [ภาษาไทย — Requirements checklist](docs/TH/42_project_requirement.md)
- [English — Requirements checklist](docs/EN/42_project_requirement.md)

### Concept Notes

| Topic | ภาษาไทย | English |
|---|---|---|
| Vagrant | [Vagrant](docs/TH/concepts/0001-Vagrant.md) | [Vagrant](<docs/EN/concepts/0001-Vagrat.md>) |
| Kubernetes | [Kubernetes (K8s)](<docs/TH/concepts/0002-Kubernetes(K8s).md>) | [Kubernetes (K8s)](<docs/EN/concepts/0002-Kubernetes(K8s).md>) |
| K3s | [K3s](<docs/TH/concepts/0003-Kubernetes(K3s).md>) | [K3s](<docs/EN/concepts/0003-Kubernetes(K3s).md>) |
| K3d | [K3d](<docs/TH/concepts/0004-Kubernetes(K3d).md>) | [K3d](<docs/EN/concepts/0004-Kubernetes(K3d).md>) |
| Helm | [Helm](docs/TH/concepts/0005-Helm.md) | [Helm](docs/EN/concepts/0005-Helm.md) |
| Argo CD | [Argo CD](docs/TH/concepts/0006-Argocd.md) | [Argo CD](docs/EN/concepts/0006-Argocd.md) |

### Implementation Notes

These notes explain the checked-in implementation mechanisms and link the explanations back to source files.

| Topic | ภาษาไทย | English |
|---|---|---|
| Repository structure | [Filestructure](docs/TH/implementation/0000-Filestructure.md) | [Filestructure](docs/EN/implementation/0000-Filestructure.md) |
| P1 — K3s provisioning | [K3s Provisioning](docs/TH/implementation/0001-P1-K3s-Provisioning.md) | [K3s Provisioning](docs/EN/implementation/0001-P1-K3s-Provisioning.md) |
| P2 — ConfigMap and sidecar | [ConfigMap and Sidecar](docs/TH/implementation/0002-P2-ConfigMap-Sidecar.md) | [ConfigMap and Sidecar](docs/EN/implementation/0002-P2-ConfigMap-Sidecar.md) |
| P2 — Service and Ingress routing | [Service and Ingress Routing](docs/TH/implementation/0003-P2-Service-Ingress-Routing.md) | [Service and Ingress Routing](docs/EN/implementation/0003-P2-Service-Ingress-Routing.md) |
| P3 — K3d and Argo CD bootstrap | [K3d and Argo CD Bootstrap](docs/TH/implementation/0004-P3-K3d-ArgoCD-Bootstrap.md) | [K3d and Argo CD Bootstrap](docs/EN/implementation/0004-P3-K3d-ArgoCD-Bootstrap.md) |
| P3 — Application exposure | [Application Exposure](docs/TH/implementation/0005-P3-Application-Exposure.md) | [Application Exposure](docs/EN/implementation/0005-P3-Application-Exposure.md) |

### Part 3 Notes

For the Part 3 setup-oriented overview, see [`p3/README.md`](p3/README.md). For the detailed behavior and limitations of the checked-in configuration, see the P3 implementation notes above.

## Suggested Reading Order

1. Start with the [repository structure note](docs/EN/implementation/0000-Filestructure.md) or its [Thai version](docs/TH/implementation/0000-Filestructure.md).
2. Read the relevant concept notes if you need background on a tool.
3. Follow the implementation notes for P1, P2, or P3 and inspect the linked source files.
4. Use each note's verification section to check behavior in a running environment.
