# Repository Structure

## Purpose

This document is a map for locating implementation code in the repository. It does not explain every file and is not a replacement for the feature-specific Implementation Notes.

This note set covers P1–P3 only. Mechanisms and limitations are documented separately so a reviewer can follow each explanation back to the code that produces the behavior.

## Repository Areas

| Area | Responsibility visible in the code | Starting points |
|---|---|---|
| `p1/` | Provisions separate K3s Server and Worker VMs, installs K3s, and passes the node token | [`p1/Vagrantfile`](../../../p1/Vagrantfile), [`server.sh`](../../../p1/scripts/server.sh), [`worker.sh`](../../../p1/scripts/worker.sh) |
| `p2/` | Provisions a K3s Server and deploys three sample apps using ConfigMap, Deployment, Service, and Ingress | [`p2/scripts/server.sh`](../../../p2/scripts/server.sh), [`p2/confs/`](../../../p2/confs/) |
| `p3/` | Creates a K3d cluster, installs Argo CD, and defines a sample app whose manifest is read from Git | [`p3/Makefile`](../../../p3/Makefile), [`wil_app.yaml`](../../../p3/confs/wil_app.yaml), [`argo.sh`](../../../p3/scripts/argo.sh) |
| `docs/TH/` and `docs/EN/` | Store Thai and English documentation, separated into Concepts, ADRs, Requirements, and Implementation Notes | `docs/TH/` and `docs/EN/` |

## Implementation Note Index

- [0001 — P1: K3s Provisioning](0001-P1-K3s-Provisioning.md) — VM creation, network identity, token handoff, and kubeconfig setup
- [0002 — P2: ConfigMap and Sidecar](0002-P2-ConfigMap-Sidecar.md) — turns ConfigMap values into web content served by Nginx through a shared volume
- [0003 — P2: Service and Ingress Routing](0003-P2-Service-Ingress-Routing.md) — selects Pods by labels and routes requests by Host/path
- [0004 — P3: K3d and Argo CD Bootstrap](0004-P3-K3d-ArgoCD-Bootstrap.md) — creates the cluster, installs Argo CD, and creates an Application
- [0005 — P3: Application Exposure](0005-P3-Application-Exposure.md) — connects host ports, the K3d load balancer, Kubernetes Service, and container ports

## Reading Principles

Implementation Notes quote only the code needed to explain a mechanism; they do not copy entire source files. The presence of configuration is not treated as proof that runtime behavior succeeded.

This draft is based on static source review only. Commands in Verification sections are proposed checks, not tests executed during this review. Where the README states a goal that is not visibly supported by configuration, the note separates that goal from the behavior that can be established from source.

## References

- [Project requirements](../42_project_requirement.md)
- [ADR-0001 — Host environment](../adrs/ADR-0001-host-environment.md)
