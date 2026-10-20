# P3 — K3d Cluster and Argo CD Bootstrap

## Context

P3 uses K3d to create a Kubernetes cluster in Docker, installs Argo CD in the `argocd` namespace, and creates the `dev` namespace for workloads managed by Argo CD. The Makefile groups these commands into targets so the setup can be invoked from one entry point.

Sources: [`p3/Makefile`](../../../p3/Makefile), [`p3/confs/wil_app.yaml`](../../../p3/confs/wil_app.yaml), [`p3/scripts/argo.sh`](../../../p3/scripts/argo.sh)

## 1. Cluster Port Mapping

**Source: `p3/Makefile:1–15`**

```makefile
ARGO_PORT = 8080
WIL_PORT = 8888

all: create-cluster dev argo

create-cluster:
	@ echo "Create K3d Cluster"
	@ k3d cluster create demo --servers 1 --agents 1 \
	--port "$(ARGO_PORT):$(ARGO_PORT)@loadbalancer" \
	--port "$(WIL_PORT):$(WIL_PORT)@loadbalancer" \
	--wait > /dev/null
```

The `all` target lists cluster creation, `dev` namespace creation, and Argo CD setup as prerequisites in that order when Make is not run in parallel mode. K3d creates a cluster named `demo` with one server and one agent and maps host ports `8080` and `8888` to the K3d load balancer.

This port mapping is a different layer from a Kubernetes Service: K3d exposes host-to-load-balancer ports, while Services inside the cluster decide how to forward traffic to Pods.

## 2. Install and Expose Argo CD

**Source: `p3/Makefile:17–36`**

```makefile
@ kubectl create namespace argocd > /dev/null 2>&1
@ kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml > /dev/null
@ kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=300s > /dev/null
@ kubectl patch svc argocd-server -n argocd --type merge -p '{"spec": {"type": "LoadBalancer", "ports": [{"name": "https", "port": 8080, "targetPort": 8080, "protocol": "TCP"}]}}'
@ kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=90s
@ until curl -sk https://localhost:8080 > /dev/null 2>&1; do sleep 1; done
```

The Makefile installs Argo CD from a remote `stable` manifest, waits for the Deployment, and patches the `argocd-server` Service to `LoadBalancer` on port 8080 to match the host-port mapping. It then waits for the HTTPS endpoint at `localhost:8080` to respond.

Because the installation URL uses `stable` rather than a pinned version/tag, installed content may change when upstream stable changes. Redirecting output and suppressing errors for namespace creation makes logs quieter but can hide useful failure details.

## 3. Log In with the Initial Admin Secret

**Source: `p3/Makefile:34–36`**

```makefile
@ PASSWORD=$$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d); \
argocd login localhost:8080 --insecure --username admin --password "$$PASSWORD"
```

The command reads the password from a Kubernetes Secret, decodes its Base64 representation, and passes it to the Argo CD CLI. `--insecure` tells the CLI to skip normal TLS certificate verification. This matches the local-testing approach in the code, but it should not be described as production-grade TLS validation.

## 4. Application Creation Has a Line-Continuation Defect

**Source: `p3/Makefile:38–47`**

```makefile
@ argocd app create wil-app \
    --repo https://github.com/lockdoor/42_inception_of_thing.git \
    --revision main
    --path p3/confs \
    --dest-server https://kubernetes.default.svc \
    --dest-namespace dev
```

The `--revision main` line does not end with `\`, so the shell ends the `argocd app create` command before `--path`, `--dest-server`, and `--dest-namespace`. The latter lines are then invoked as separate commands rather than arguments to `argocd app create`. This is a source-visible defect that should be fixed/tested before claiming the `argo` target completes successfully.

`p3/scripts/argo.sh:7–11` contains another Application creation command with a different repository URL and path. It sets `--path ./confs/wil_app.yaml`, which points to a single manifest file, while `argocd app create --path` expects an application directory; for a manifest file, verify the CLI `--file` option instead. The reviewer should identify which entry point was actually tested rather than assume both procedures are equivalent.

In `p3/confs/wil_app.yaml:5,30`, namespace fields are commented out while the Makefile sets `--dest-namespace dev`. Git history contains a commit titled `fixed: p3 will app namespace`, whose diff removes the hard-coded namespaces ([commit 7a8c9b9](https://github.com/lockdoor/42_inception_of_thing/commit/7a8c9b98b79abca29d00ea47ba5d4a96bb476b96)). This supports the interpretation that the destination namespace was intended to determine the workload namespace, but it is not runtime test evidence.

## 5. Git Change Detection Is Not the Same as Auto-Sync

The README says Argo CD will deploy after changes are pushed to GitHub. However, the Makefile's `argocd app create` command does not set `--sync-policy automated`, and `wil_app.yaml` is a Deployment/Service manifest, not an Argo CD Application manifest containing `spec.syncPolicy.automated`.

Therefore, the visible source does not establish that automatic sync is enabled, even if Argo CD can detect a new revision. Argo CD documentation says an automated sync policy must be configured explicitly ([Automated Sync Policy](https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/)).

The command `kubectl wait --for=condition=synced app/wil-app -n dev` has two concerns. The Application resource normally lives in Argo CD's control-plane namespace (often `argocd`) unless application namespaces are configured differently; `--dest-namespace dev` sets the destination namespace for workloads, not where the Application resource itself is stored. Also, sync status is exposed as `status.sync.status`, so do not assume it behaves like a normal Kubernetes condition on a Deployment. Verify the Application namespace and the correct way to wait for sync before relying on this command.

## Edge Cases and Limitations

- The Application-creation command in the Makefile has an incomplete line continuation, so the source alone does not establish that `make all` works.
- The remote `stable` manifest does not pin the Argo CD version.
- Namespace-creation errors are suppressed; verify namespaces and resources separately.
- The README describes intended behavior, but the visible configuration does not clearly enable auto-sync.
- `p3/confs/wil_app.yaml` must be read from the exact Git revision/path configured in the Application; a local working-tree change does not itself prove Argo CD sees that change.

## Verification

```bash
k3d cluster list
kubectl get ns argocd dev
kubectl get deploy,svc -n argocd
argocd app list
argocd app get wil-app
kubectl get applications.argoproj.io -A -o yaml
```

Inspect the Application's source repo/path/revision, destination namespace, sync policy, and actual sync status before confirming the workflow. The README alone is not evidence that automatic deployment has been verified.

## References

- [Argo CD `argocd app create`](https://argo-cd.readthedocs.io/en/latest/user-guide/commands/argocd_app_create/)
- [Argo CD Automated Sync Policy](https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/)
- [Git history: P3 application namespace change](https://github.com/lockdoor/42_inception_of_thing/commit/7a8c9b98b79abca29d00ea47ba5d4a96bb476b96)
- Sources: [`Makefile`](../../../p3/Makefile), [`wil_app.yaml`](../../../p3/confs/wil_app.yaml), [`argo.sh`](../../../p3/scripts/argo.sh), [`README.md`](../../../p3/README.md)
