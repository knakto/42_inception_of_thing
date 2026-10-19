# P3 — Exposing the Application Through K3d and a LoadBalancer Service

## Context

ใน P3 host machine ต้องเข้าถึงทั้ง Argo CD และเว็บตัวอย่างจาก K3d cluster ได้ Makefile จึง map host ports ไปยัง K3d load balancer ขณะที่ Kubernetes Service ของแต่ละแอปกำหนดเส้นทางจาก Service port ไปยัง port ใน Pod อีกชั้นหนึ่ง

Source: [`p3/Makefile`](../../../p3/Makefile), [`p3/confs/wil_app.yaml`](../../../p3/confs/wil_app.yaml)

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

การ map พอร์ตนี้ทำให้ host ports `8080` และ `8888` ถูกส่งไปยังพอร์ตเดียวกันบน K3d load balancer ตัวเลขด้านซ้ายคือพอร์ตบน host; ตัวเลขด้านขวาคือพอร์ตที่เปิดผ่าน load balancer ตามรูปแบบของ `k3d cluster create --port`

การ map นี้ไม่สร้าง Kubernetes Service และไม่ได้แปลว่า container ทุกตัวเปิดพอร์ตเหล่านี้เอง ต้องมี Service/ปลายทางที่สอดคล้องกันด้วย

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

Deployment ขอให้มี Pod หนึ่งตัว และ label `app: wil-app` เป็นตัวเชื่อมให้ Service ค้นหา Pod ได้ ส่วน `containerPort: 8888` บันทึกพอร์ตที่คาดว่า container ใช้ ไม่ได้เปิด port forwarding ด้วยตัวเอง และไม่รับประกันว่า process ภายใน image ฟังพอร์ตนั้นจริง ต้องตรวจ image/process หรือทดลองส่ง request

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

ค่านี้มีหน้าที่ต่างกัน:

- `port: 8888` เป็นพอร์ตของ Kubernetes Service ที่ client ภายใน cluster ใช้เรียก Service
- `targetPort: 8888` เป็นพอร์ตปลายทางของ container/Pod ที่รับ traffic
- `nodePort: 30000` ขอใช้พอร์ตของ node สำหรับกลไก NodePort ที่ Service รองรับ
- `type: LoadBalancer` ขอให้ cluster จัดเตรียม load-balancer exposure สำหรับ Service; ใน K3d ต้องดูว่าการเข้าถึงจาก host เชื่อมกับ port mapping ที่ตั้งไว้หรือไม่

เพราะ `k3d cluster create` map host port `8888` ไปยัง load balancer port `8888` โค้ดแสดงเจตนาให้เรียกเว็บผ่าน `localhost:8888` ไม่ใช่ `localhost:30000` การระบุ NodePort ใน manifest ไม่ได้สร้าง host-port mapping `30000:30000` ใน Makefile

## 4. Argo CD Uses the Other Port

**Source: `p3/Makefile:27–36`**

```makefile
@ kubectl patch svc argocd-server -n argocd --type merge -p '{"spec": {"type": "LoadBalancer", "ports": [{"name": "https", "port": 8080, "targetPort": 8080, "protocol": "TCP"}]}}'
@ until curl -sk https://localhost:8080 > /dev/null 2>&1; do sleep 1; done
```

Service ของ Argo CD ถูก patch ให้ใช้ port 8080 เพื่อจับคู่กับ host-to-load-balancer mapping ที่ประกาศไว้ก่อนหน้า จากนั้น Makefile รอจน `https://localhost:8080` ตอบสนอง โค้ดใช้ `curl -k` จึงข้ามการตรวจ certificate; เป็นการตรวจว่ามี HTTPS endpoint ตอบกลับ ไม่ใช่การยืนยันความถูกต้องของ certificate

## Edge Cases and Limitations

- `containerPort` เป็น metadata; แอปต้องฟัง port นั้นจริง
- Host port mapping ของ K3d กับ Service port เป็นคนละชั้นและต้องเข้าคู่กัน
- `nodePort: 30000` อยู่ใน manifest แต่ไม่มี mapping ของ host port 30000 ในคำสั่งสร้าง cluster
- Service selector ต้องตรงกับ `app: wil-app`; หากไม่ตรงจะไม่มี endpoint ที่ตรงกับ Pod
- `curl -k` เหมาะกับ local verification ที่ตั้งใจข้าม certificate check แต่ไม่ควรใช้ผลนี้อ้างว่า TLS identity ถูกตรวจสอบแล้ว

## Verification

```bash
kubectl get svc wil-app-service -n dev -o yaml
kubectl get endpointslices -n dev -l kubernetes.io/service-name=wil-app-service
kubectl get pods -n dev -l app=wil-app -o wide
curl -i http://localhost:8888/
curl -k -I https://localhost:8080/
```

ตรวจค่า `port`, `targetPort`, Service type, Pod endpoints และผลตอบกลับจริง ถ้า `localhost:8888` ใช้งานไม่ได้ ให้แยกตรวจ host port mapping, Service/endpoints และพอร์ตที่ process ใน container ฟังอยู่ แทนการสรุปว่าเป็นปัญหาเดียวกันทั้งหมด

## References

- [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [K3d port mapping](https://k3d.io/stable/usage/exposing_services/)
- Source: [`Makefile`](../../../p3/Makefile), [`wil_app.yaml`](../../../p3/confs/wil_app.yaml)
