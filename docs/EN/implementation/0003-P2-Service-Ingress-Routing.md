# P2 — Service Discovery and Host-Based Ingress Routing

## Context

P2 has three sample apps with Nginx listening on port 80 inside each Pod. Requests should reach the correct app based on the HTTP Host supplied by the client. Kubernetes Service provides a stable destination for matching Pods, while Ingress declares HTTP routing rules from Host/path to a Service.

Sources: [`iot-service.yaml`](../../../p2/confs/iot-service.yaml), [`iot-ingress.yaml`](../../../p2/confs/iot-ingress.yaml), [`kustomization.yaml`](../../../p2/confs/kustomization.yaml), [`p2/scripts/server.sh`](../../../p2/scripts/server.sh)

## 1. A Service Selects Pods by Labels

**Source: `p2/confs/iot-service.yaml:3–34`**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: svc-app2
spec:
  selector:
    app: iot-app2
  ports:
    - port: 80
      targetPort: 80
```

`svc-app2` selects Pods with label `app: iot-app2` and forwards traffic from Service port 80 to target port 80 on the selected endpoints. A Service does not select a Deployment directly; the Deployment creates Pods with matching labels, and Kubernetes creates EndpointSlices from Pods that match the Service selector.

If the label changes on one side without a matching change on the other, the Deployment can still have Pods while the Service has no endpoints to receive traffic.

## 2. Ingress Maps Host and Path to a Service

**Source: `p2/confs/iot-ingress.yaml:6–46`**

```yaml
spec:
  rules:
    - host: app1.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: svc-app1
                port:
                  number: 80
    - host: app2.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: svc-app2
                port:
                  number: 80
    - host: app3.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: svc-app3
                port:
                  number: 80
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: svc-app3
                port:
                  number: 80
```

The first three rules choose a Service by Host. `path: /` with `pathType: Prefix` matches paths under `/` according to Ingress prefix matching. The final rule omits `host`, so it acts as the rule for HTTP requests not matched by the Host-specific rules; in this manifest, those requests go to `svc-app3`.

This fallback is explicitly configured in the source; it is not created by the browser or DNS. Overriding the Host header is a way to test HTTP routing directly.

An Ingress resource is not an Ingress Controller. The resource declares desired rules; a controller in the cluster must process them and handle traffic. This manifest does not explicitly set `ingressClassName`, so the controller/default class actually processing the rule must be verified in the cluster.

## 3. Kustomize Collects the Resources

**Source: `p2/confs/kustomization.yaml:1–11`**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - iot-configmap.yaml
  - iot-deployment.yaml
  - iot-service.yaml
  - iot-ingress.yaml
```

The Kustomization file lists the resource set to process together, allowing the script to use `kubectl apply -k` on a directory rather than applying each manifest separately.

**Source: `p2/scripts/server.sh:13–16`**

```bash
ln -s /mnt/confs /home/vagrant/confs

# Apply kustomaized manifest
kubectl apply -k ./confs
```

Vagrant mounts `p2/confs` at `/mnt/confs`; the script creates `/home/vagrant/confs` as a symlink to that mount and then runs Kustomize with the relative path `./confs`. Resolution of that path depends on the provisioner's working directory, so it should be checked at runtime if the directory is not found.

## Edge Cases and Limitations

- If a Service selector does not match Pod labels, the Service may have no endpoints.
- A Host outside the three configured names falls through to `svc-app3` according to this manifest.
- `ingressClassName` is not set; handling depends on the controller/default class actually configured in the cluster.
- An Ingress resource alone does not prove the controller is ready or that DNS for `app1.com`, `app2.com`, and `app3.com` points to the VM.
- Ingress, Service, and referenced Pods must be in compatible namespaces. These manifests omit namespace fields, so they use the default namespace unless the deployment configuration adds a namespace.

## Verification

```bash
kubectl get ingress iot-apps-ingress -o yaml
kubectl get ingressclass
kubectl get svc svc-app1 svc-app2 svc-app3
kubectl get pods --show-labels
kubectl get endpointslices -l kubernetes.io/service-name=svc-app2

curl -i -H 'Host: app1.com' http://192.168.56.110/
curl -i -H 'Host: app2.com' http://192.168.56.110/
curl -i -H 'Host: unknown.example' http://192.168.56.110/
```

Expected: `app1.com` and `app2.com` route to their respective Services; an unknown Host uses the `svc-app3` fallback, provided an Ingress Controller is actually processing this rule.

## References

- [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/)
- Sources: [`iot-service.yaml`](../../../p2/confs/iot-service.yaml), [`iot-ingress.yaml`](../../../p2/confs/iot-ingress.yaml), [`kustomization.yaml`](../../../p2/confs/kustomization.yaml)
