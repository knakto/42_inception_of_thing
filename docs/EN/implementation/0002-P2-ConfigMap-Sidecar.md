# P2 — Generating Per-Pod Web Content with a ConfigMap and Sidecar

## Context

P2 runs three sample web apps using Nginx to serve static files. The page text comes from a ConfigMap, while another container writes `index.html` into a volume shared by both containers.

The observable effect is that text is separated from the web image and page content is generated at runtime for each Pod. This appears to avoid baking each page's content directly into the Nginx image, but the source does not explicitly state the original design rationale, so that should not be presented as a confirmed requirement.

Sources: [`iot-configmap.yaml`](../../../p2/confs/iot-configmap.yaml), [`iot-deployment.yaml`](../../../p2/confs/iot-deployment.yaml)

## 1. The ConfigMap Supplies the Text

**Source: `p2/confs/iot-configmap.yaml:1–8`**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: iot-configmap
data:
  app1: "Hello from app1"
  app2: "Hello from app2"
  app3: "Hello from app3"
```

One ConfigMap stores three values. When mounted as a volume at `/etc/config`, each key becomes a file. For example, key `app1` is read from `/etc/config/app1` inside a container mounting this volume.

## 2. Two Containers Share a Volume

**Source: `p2/confs/iot-deployment.yaml:17–47` (App 1)**

```yaml
volumes:
  - name: shared-data
    emptyDir: {}
  - name: config-volume
    configMap:
      name: iot-configmap
containers:
  - name: nginx
    image: "nginx:stable-alpine3.24-perl"
    ports:
      - containerPort: 80
    volumeMounts:
      - name: shared-data
        mountPath: /usr/share/nginx/html
initContainers:
  - name: alpine
    image: "alpine:3.23.5"
    restartPolicy: Always
    volumeMounts:
      - name: shared-data
        mountPath: /pod-data
      - name: config-volume
        mountPath: /etc/config
    command:
      - /bin/sh
      - -c
      - |
        while true; do
          echo "$(cat /etc/config/app1), From $HOSTNAME" > /pod-data/index.html;
          sleep 10;
        done;
```

The `shared-data` volume uses `emptyDir`, giving one Pod a temporary directory that both containers can access:

1. Alpine reads `/etc/config/app1`.
2. Alpine writes the text and `$HOSTNAME` to `/pod-data/index.html`.
3. Nginx sees the same file at `/usr/share/nginx/html/index.html` because both paths mount the same volume.
4. The loop overwrites the file every 10 seconds.

Kubernetes normally sets `HOSTNAME` to the Pod name. This can help distinguish which replica answered, but the actual output should be checked in the cluster before relying on it as a test result.

## 3. Why `restartPolicy: Always` Matters

The container is listed under `initContainers`, but has a container-level `restartPolicy: Always`. Kubernetes treats this as a **restartable sidecar init container**, which continues running alongside the main container after Pod startup. It is not a regular init container that must finish before application containers start.

This matters because the loop is intended to keep rewriting the file every 10 seconds. A regular init container that never exits would prevent the main application container from starting as intended.

Support for container-level `restartPolicy` depends on the Kubernetes version/feature gate. Kubernetes documents sidecar containers as stable from v1.33 and enabled by default from v1.29 ([Sidecar Containers](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/)). P2 does not pin the K3s version, so check the cluster version before relying on this behavior.

## 4. Replicas Are Independent Pods

In the App 2 Deployment (`iot-deployment.yaml:63–103`), `replicas: 3` asks the Deployment controller to maintain three Pods. Each Pod has its own `emptyDir`, so each has its own `index.html`; the Alpine container writes its own Pod hostname. Apps 1 and 3 specify `replicas: 1`.

Replica count is not directly driven by HTTP request volume. The Deployment controller reconciles the requested replica count; a Service does not create extra Pods when traffic increases.

## Edge Cases and Limitations

- `emptyDir` follows the Pod lifecycle; data written there is not persistent when a Pod is deleted and recreated.
- The ConfigMap separates text from the image, but the script does not validate that a key exists before reading it.
- Content is overwritten every 10 seconds; there is no explicit handling for read/write errors.
- Apps 1/2/3 repeat the same template with different keys, replica counts, and labels; changing one block does not change the other two.
- The K3s/Kubernetes version is not pinned even though the manifest relies on sidecar-container semantics.

## Verification

```bash
kubectl get configmap iot-configmap -o yaml
kubectl get deployments
kubectl get pods -l app=iot-app2 -o wide
kubectl describe pod <pod-name>
kubectl exec <pod-name> -c nginx -- cat /usr/share/nginx/html/index.html
kubectl exec <pod-name> -c alpine -- cat /pod-data/index.html
kubectl get --raw /version
```

Compare the file from both mount paths in the same Pod and confirm that App 2 has three Pods in the actual cluster. These are proposed checks, not commands executed during this review.

## References

- [Kubernetes ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Kubernetes volumes: `emptyDir`](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir)
- [Kubernetes sidecar containers](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/)
- Sources: [`iot-configmap.yaml`](../../../p2/confs/iot-configmap.yaml), [`iot-deployment.yaml`](../../../p2/confs/iot-deployment.yaml)
