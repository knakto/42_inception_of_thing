# P2 — Generating Per-Pod Web Content with ConfigMap and Sidecar

## Context

P2 สร้างเว็บตัวอย่างสามตัวจาก Deployment ที่ใช้ Nginx ให้บริการไฟล์ static โดยเนื้อหาในหน้าเว็บมาจาก ConfigMap และมี container อีกตัวเขียน `index.html` ลง volume ที่ทั้งสอง container ใช้ร่วมกัน

สิ่งที่ implementation นี้ทำคือแยกข้อมูลข้อความออกจาก image ของเว็บ และสร้างเนื้อหาของแต่ละ Pod ขณะทำงาน แทนการใส่ไฟล์หน้าเว็บแต่ละเวอร์ชันเข้าไปใน Nginx image โดยตรง เหตุผลเชิงธุรกิจหรือข้อกำหนดที่ทำให้เลือกวิธีนี้ไม่ได้ระบุไว้ชัดเจนในโค้ด จึงไม่ควรอ้างว่าเป็นเหตุผลที่ยืนยันแล้ว

Source: [`iot-configmap.yaml`](../../../p2/confs/iot-configmap.yaml), [`iot-deployment.yaml`](../../../p2/confs/iot-deployment.yaml)

## 1. ConfigMap Supplies the Text

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

หนึ่ง ConfigMap เก็บข้อความสามค่า แต่ละ key ถูกใช้เป็นชื่อไฟล์เมื่อ mount ConfigMap เป็น volume ที่ `/etc/config` ตัวอย่างเช่น key `app1` อ่านได้จาก `/etc/config/app1` ภายใน container ที่ mount volume นี้

## 2. The Two Containers Share a Volume

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

Volume `shared-data` ใช้ `emptyDir` ซึ่งสร้างพื้นที่ว่างสำหรับอายุของ Pod หนึ่งตัว และทำให้ container ที่ mount volume เดียวกันอ่านเขียนไฟล์ชุดเดียวกันได้:

1. Alpine อ่านค่าจาก `/etc/config/app1`
2. Alpine เขียนข้อความและ `$HOSTNAME` ลง `/pod-data/index.html`
3. Nginx เห็นไฟล์เดียวกันที่ `/usr/share/nginx/html/index.html` เพราะเป็น volume เดียวกันแต่ mount คนละ path
4. loop ทำงานทุก 10 วินาที จึงเขียนทับเนื้อหาเดิมเป็นระยะ

Kubernetes กำหนด `HOSTNAME` ให้ container ภายใน Pod โดยปกติเป็นชื่อ Pod ดังนั้นข้อความช่วยให้แยกได้ว่าคำตอบมาจาก replica ใด อย่างไรก็ตามต้องตรวจผลจริงใน cluster หากต้องใช้พฤติกรรมนี้เป็นเกณฑ์ทดสอบ

## 3. Why `restartPolicy: Always` Matters

Container นี้อยู่ใต้ `initContainers` แต่กำหนด `restartPolicy: Always` ที่ระดับ container การตั้งค่านี้ทำให้ Kubernetes จัดการมันเป็น **restartable sidecar init container** ซึ่งอยู่ทำงานหลังเริ่ม Pod ต่อไปพร้อม container หลัก แทน init container ปกติที่ทำงานจนจบก่อนเริ่ม application containers

รูปแบบนี้จำเป็นต่อ loop ที่ต้องเขียนไฟล์ซ้ำทุก 10 วินาที หากเป็น init container ปกติ loop จะไม่จบและการเริ่ม container หลักจะไม่เดินหน้าตามต้องการ

การรองรับ container-level `restartPolicy` ขึ้นกับ Kubernetes version/feature gate โดยเอกสาร Kubernetes ระบุว่า sidecar containers เป็น stable ตั้งแต่ v1.33 และ feature เปิดใช้โดย default ตั้งแต่ v1.29 [Sidecar Containers](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/). สคริปต์ติดตั้ง K3s ใน P2 ไม่ได้ pin เวอร์ชันไว้ จึงควรตรวจ cluster version ก่อนพึ่งพาพฤติกรรมนี้

## 4. Replicas Are Independent Pods

ใน Deployment ของ App 2 (`iot-deployment.yaml:63–103`) `replicas: 3` ขอให้ Deployment controller รักษา Pod สามตัว แต่ละ Pod มี `emptyDir` ของตัวเอง จึงมี `index.html` แยกกัน และ Alpine ในแต่ละ Pod เขียน hostname ของ Pod ตัวเอง ส่วน App 1 และ App 3 กำหนด `replicas: 1`

จำนวน replica ไม่ได้ขึ้นกับจำนวน HTTP requests โดยตรง การเปลี่ยนจำนวน Pod เกิดจาก desired state ของ Deployment และการทำงานของ controller ไม่ใช่ Service ที่เพิ่ม Pod ตาม traffic

## Edge Cases and Limitations

- `emptyDir` อยู่ตาม lifecycle ของ Pod; เมื่อ Pod ถูกลบและสร้างใหม่ ไฟล์ที่สร้างไว้ไม่ใช่ persistent data
- ConfigMap แยกข้อมูลข้อความจาก image แต่ไม่มี validation ใน script ว่า key ที่อ่านมีอยู่จริงหรือไม่
- เนื้อหาถูกเขียนทับทุก 10 วินาที ไม่มี logic จัดการ error จาก `cat` หรือเขียนไฟล์
- App 1/2/3 ใช้ template ซ้ำกันต่างกันที่ key, replica count และ label; การแก้หนึ่งชุดไม่ได้แปลว่าจะเปลี่ยนอีกสองชุด
- ไม่ได้ pin K3s/Kubernetes version แม้ manifest จะพึ่งพาความสามารถของ sidecar container

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

เปรียบเทียบไฟล์จากสอง mount path ใน Pod เดียวกัน และตรวจว่า App 2 มีสาม Pod ตามสถานะจริง คำสั่งเหล่านี้เป็นแนวทางตรวจ ไม่ใช่ผลทดสอบที่รันแล้ว

## References

- [Kubernetes ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Kubernetes volumes: `emptyDir`](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir)
- [Kubernetes sidecar containers](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/)
- Source: [`iot-configmap.yaml`](../../../p2/confs/iot-configmap.yaml), [`iot-deployment.yaml`](../../../p2/confs/iot-deployment.yaml)
