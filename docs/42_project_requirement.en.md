# 42 Inception of Things — Project Requirements

> Checklist based on the *Inception of Things* subject, version 4.1. Mandatory requirements and recommendations are distinguished below.

## General Requirements

- [ ] Complete the project in a Virtual Machine environment.
- [ ] Keep the project and configuration files in the repository using the required structure.
- [ ] Place the mandatory parts in `p1/`, `p2/`, and `p3/` at the repository root.
- [ ] Place necessary scripts in `scripts/` and configuration files in `confs/` as appropriate.
- [ ] Complete Part 1, Part 2, and Part 3 in the specified order.
- [ ] Ensure the project can be evaluated on the evaluator's machine using the files in the repository.

## Part 1 — K3s and Vagrant

- [ ] Create two Virtual Machines using Vagrant.
- [ ] Use the latest stable version of the chosen Linux distribution.
- [ ] Name the machines and hostnames using a team member's login followed by `S` for the Server and `SW` for the ServerWorker.
- [ ] Assign IP `192.168.56.110` to the Server and `192.168.56.111` to the ServerWorker.
- [ ] Use a dedicated network interface for the assigned IP address on each machine.
- [ ] Allow passwordless SSH access to both Virtual Machines.
- [ ] Install K3s in Server/controller mode on the first machine.
- [ ] Install K3s in Agent mode on the second machine and join it to the Server's cluster.
- [ ] Install and use `kubectl` to manage Kubernetes.
- [ ] Specify the source of the VM box in the Vagrant configuration using `config.vm.box_url` pointing to a distribution-published image, or use a self-built box.

## Part 2 — K3s and Three Web Applications

- [ ] Prepare one Virtual Machine using the latest stable version of the chosen Linux distribution.
- [ ] Install K3s in Server mode on that machine.
- [ ] Deploy three chosen web applications to the K3s instance.
- [ ] Route requests to the applications through IP `192.168.56.110` according to the HTTP `Host` header.
- [ ] Requests with `Host: app1.com` must reach Application 1.
- [ ] Requests with `Host: app2.com` must reach Application 2.
- [ ] Requests with any other Host must reach Application 3 by default.
- [ ] Configure Application 2 with 3 replicas.
- [ ] Be prepared to show the Ingress during evaluation.

## Part 3 — K3d and Argo CD

- [ ] Install K3d on the Virtual Machine.
- [ ] Prepare Docker and any other packages/tools required by K3d.
- [ ] Provide a script to install the necessary packages and tools for use during evaluation.
- [ ] Create a Kubernetes namespace named `argocd` for Argo CD.
- [ ] Create a Kubernetes namespace named `dev` for the application.
- [ ] Create a public GitHub repository for the configuration; its name must include the login of at least one team member.
- [ ] Configure Argo CD to automatically deploy the application into the `dev` namespace from the public GitHub repository.
- [ ] Provide two different versions of the application.
- [ ] Use either the sample application `wil42/playground` or an application developed by the team.
- [ ] If using a custom application, publish its Docker image to a public Docker Hub repository and tag the versions `v1` and `v2`.
- [ ] If using a custom application, make the `v1` and `v2` versions visibly different.
- [ ] Change the application version through the public GitHub repository and verify that Argo CD deploys the updated version successfully.

## Submission and Evaluation

- [ ] Keep `p1/`, `p2/`, and `p3/` at the repository root.
- [ ] Keep the files needed to set up and verify the system in the repository.
- [ ] Check directory and file names before submission.
- [ ] Verify that setup and demonstrations can be reproduced on the evaluator's machine.

## Bonus — GitLab (Optional)

- [ ] Place the bonus work in `bonus/` at the repository root.
- [ ] Run GitLab locally.
- [ ] Configure GitLab to work with the cluster.
- [ ] Create a namespace named `gitlab`.
- [ ] Make the complete Part 3 workflow work with local GitLab instead of GitHub.

## Recommendations from the Subject (Resource Guidance, Not a Hard Requirement)

- The subject recommends keeping VM resource allocation to a minimum, giving 1 CPU and 512 MB (or 1024 MB) RAM as examples.
- Consult the documentation for K3s, K3d, Kubernetes, and the selected tools as needed.
