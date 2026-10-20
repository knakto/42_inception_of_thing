# P2 — Service Discovery and Host-Based Ingress Routing

## Context

P2 มีแอปตัวอย่างสามตัวที่ใช้ Nginx ฟังพอร์ต 80 ภายใน Pod โดยต้องส่ง request ไปยังแอปที่ถูกต้องตาม HTTP Host ที่ผู้เรียกใช้ระบุ Kubernetes Service ทำหน้าที่สร้างปลายทางที่คงที่ให้ Pod ส่วน Ingress ประกาศกฎ HTTP routing จาก Host/path ไปยัง Service

Source: [`iot-service.yaml`](../../../p2/confs/iot-service.yaml), [`iot-ingress.yaml`](../../../p2/confs/iot-ingress.yaml), [`kustomization.yaml`](../../../p2/confs/kustomization.yaml), [`p2/scripts/server.sh`](../../../p2/scripts/server.sh)

## 1. Service Selects Pods by Labels

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

Service `svc-app2` เลือก Pod ที่มี label `app: iot-app2` และส่ง traffic ที่ Service port 80 ไปยัง target port 80 ของ Pod ที่เป็น endpoint ปัจจุบัน Service ไม่ได้เลือก Deployment โดยตรง; Deployment สร้าง Pod ที่มี label ตรงกัน และ Kubernetes สร้าง EndpointSlices จาก Pod ที่ match กับ Service selector

จุดนี้ทำให้การเปลี่ยนชื่อ label ข้างใดข้างหนึ่งโดยไม่ปรับอีกข้างอาจทำให้ Service ไม่มี endpoint แม้ Deployment ยังมี Pod อยู่ก็ตาม

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

กฎสามข้อแรกใช้ Host header เลือก Service; path `/` ที่ใช้ `pathType: Prefix` จับ path ที่เริ่มด้วย `/` ตาม semantics ของ Ingress Prefix matching ส่วนกฎสุดท้ายไม่มี `host` จึงเป็นกฎสำหรับ request HTTP ที่ไม่ match Host-specific rules และใน manifest นี้ส่งไป `svc-app3`

นี่คือ fallback ที่ประกาศไว้จริงใน code ไม่ใช่พฤติกรรมของ Browser หรือ DNS โดยตัวมันเอง ส่วนการทดสอบด้วยการ override Host header เป็นการทดสอบ routing ของ HTTP request โดยตรง

Ingress resource ไม่ใช่ Ingress Controller ตัว resource ระบุ desired rules; controller ใน cluster ต้องอ่าน rules แล้วรับ traffic ให้ได้ Manifest นี้ไม่ได้กำหนด `ingressClassName` ไว้ชัดเจน จึงต้องตรวจว่ามี controller ใดรับผิดชอบกฎนี้จริงใน cluster

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

ไฟล์ Kustomization ระบุ resource set ที่จะนำมาประมวลผลร่วมกัน จึงทำให้สคริปต์ใช้ `kubectl apply -k` กับ directory แทนการเรียก apply แยกทุก manifest

**Source: `p2/scripts/server.sh:13–16`**

```bash
ln -s /mnt/confs /home/vagrant/confs

# Apply kustomaized manifest
kubectl apply -k ./confs
```

Vagrant mount `p2/confs` ที่ `/mnt/confs` จากนั้น script สร้าง symlink `/home/vagrant/confs` ไปยังตำแหน่งนั้น แล้วเรียก Kustomize ผ่าน relative path `./confs` การ resolve path ของคำสั่งนี้ขึ้นกับ working directory ที่ provisioner ใช้ จึงควรตรวจจาก runtime หากคำสั่งหา directory ไม่พบ

## Edge Cases and Limitations

- ถ้า selector ของ Service ไม่ตรง labels ของ Pod, Service อาจไม่มี endpoints ให้ส่ง traffic
- Host ที่ไม่ตรงกับสาม Host ที่กำหนดจะเข้า rule ที่ไม่มี Host และถูกส่งไป `svc-app3` ตาม manifest นี้
- ไม่มี `ingressClassName` ใน resource; การประมวลผลจึงขึ้นกับ controller/default class ที่ cluster ใช้จริง
- การมี Ingress resource ไม่ได้พิสูจน์ว่า controller พร้อมรับ traffic หรือว่า DNS ของ `app1.com`/`app2.com`/`app3.com` ชี้มาที่ VM แล้ว
- ต้องให้ Ingress, Service และ Pod อยู่ใน namespace ที่อ้างอิงกันได้; manifest ในชุดนี้ไม่ได้ระบุ namespace จึงใช้ namespace default เว้นแต่มีการกำหนดเพิ่มในขั้น deploy

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

คาดหวังว่า Host `app1.com`/`app2.com` ไปยัง Service ที่สอดคล้องกัน และ Host อื่นจะใช้ fallback `svc-app3` หาก Ingress Controller รับกฎนี้อยู่จริง

## References

- [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/)
- Source: [`iot-service.yaml`](../../../p2/confs/iot-service.yaml), [`iot-ingress.yaml`](../../../p2/confs/iot-ingress.yaml), [`kustomization.yaml`](../../../p2/confs/kustomization.yaml)
