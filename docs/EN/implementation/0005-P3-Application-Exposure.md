# P3 — Exposing the Application Through K3d and a LoadBalancer Service

## Context

In P3, the host must reach both Argo CD and the sample web application running inside K3d. The Makefile maps host ports to the K3d load balancer, while the Kubernetes Service defines a separate forwarding step from the Service port to the Pod's port.

Sources: [`p3/Makefile`](../../../p3/Makefile), [`p3/confs/wil_app.yaml`](../../../p3/confs/wil_app.yaml)

## 1. Host-to-K3d Port Mapping

**Source: `p3/Makefile:1–15`**

```makefile
ARGO_PORT = 8080
WIL_PORT = 8888

@ k3d cluster create demo --servers 1 --agents 1 \
    --port "$(ARGO_PORT):$(ARGO_PORT)@loadbalancer" \
    --port "$(WIL_PORT):$(WIL_PORT)@loadbalancer" \
    --wait > /dev/null
```

This maps host ports `8080` and `8888` to the same ports exposed through the K3d load balancer. The left side is the host port; the right side is the port on the load balancer, following the `k3d cluster create --port` syntax.

This mapping does not create a Kubernetes Service and does not make every container listen on these ports. A matching Service and destination still have to exist inside the cluster.

## 2. The Application Listens on 8888

**Source: `p3/confs/wil_app.yaml:1–22`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wil-app-deployment
  labels:
    app: wil-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wil-app
  template:
    metadata:
      labels:
        app: wil-app
    spec:
      containers:
        - name: wil-app
          image: wil42/playground:v2
          ports:
            - containerPort: 8888
```

The Deployment requests one Pod, and label `app: wil-app` connects the Pod to the Service selector. `containerPort: 8888` documents the port expected to be used by the container; it does not create host port forwarding and does not guarantee that the process inside the image actually listens there. That still needs to be checked by inspecting the image/process or sending a request.

## 3. Service Port, Target Port, and NodePort

**Source: `p3/confs/wil_app.yaml:26–39`**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: wil-app-service
spec:
  type: LoadBalancer
  selector:
    app: wil-app
  ports:
    - protocol: TCP
      port: 8888
      targetPort: 8888
      nodePort: 30000
```

These fields have different responsibilities:

- `port: 8888` is the Kubernetes Service port clients use inside the cluster.
- `targetPort: 8888` is the destination port on the selected Pod/container.
- `nodePort: 30000` requests a node port for the Service's NodePort mechanism.
- `type: LoadBalancer` asks the cluster to provide load-balancer exposure for the Service; with K3d, verify how that exposure connects to the configured host-port mapping.

Because `k3d cluster create` maps host port `8888` to load-balancer port `8888`, the code indicates that the intended web entry point is `localhost:8888`, not `localhost:30000`. Declaring a NodePort in the manifest does not create a `30000:30000` host-port mapping in the Makefile.

## 4. Argo CD Uses the Other Port

**Source: `p3/Makefile:27–36`**

```makefile
@ kubectl patch svc argocd-server -n argocd --type merge -p '{"spec": {"type": "LoadBalancer", "ports": [{"name": "https", "port": 8080, "targetPort": 8080, "protocol": "TCP"}]}}'
@ until curl -sk https://localhost:8080 > /dev/null 2>&1; do sleep 1; done
```

The Argo CD Service is patched to use port 8080 to match the host-to-load-balancer mapping. The Makefile then waits for `https://localhost:8080` to respond. Since the command uses `curl -k`, it skips certificate validation: this checks that an HTTPS endpoint responds, not that its certificate identity was verified.

## Edge Cases and Limitations

- `containerPort` is metadata; the app must actually listen on that port.
- K3d host mapping and Kubernetes Service ports are separate layers and must line up.
- `nodePort: 30000` appears in the manifest, but the cluster-creation command does not map host port 30000.
- The Service selector must match `app: wil-app`, otherwise it will not select the Pod.
- `curl -k` is useful for the configured local check but does not prove that TLS identity was validated.

## Verification

```bash
kubectl get svc wil-app-service -n dev -o yaml
kubectl get endpointslices -n dev -l kubernetes.io/service-name=wil-app-service
kubectl get pods -n dev -l app=wil-app -o wide
curl -i http://localhost:8888/
curl -k -I https://localhost:8080/
```

Check the Service type, `port`, `targetPort`, endpoints, and actual response. If `localhost:8888` fails, separate the investigation into host mapping, Service/endpoints, and the port listened to by the process inside the container rather than treating them as one setting.

## References

- [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [K3d port mapping](https://k3d.io/stable/usage/exposing_services/)
- Sources: [`Makefile`](../../../p3/Makefile), [`wil_app.yaml`](../../../p3/confs/wil_app.yaml)
