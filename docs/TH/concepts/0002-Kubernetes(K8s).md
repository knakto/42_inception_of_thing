# Kubernetes (K8s)

## Overview

Kubernetes (K8s) เป็น Open-Source Platform สำหรับจัดการ Containerized Applications โดยช่วยควบคุมการทำงานของ Containers ภายใน Cluster เช่น การ Deploy แอปพลิเคชัน การปรับจำนวน Replicas การกระจาย Workloads ไปยัง Nodes และการจัดการสถานะของแอปพลิเคชัน

### Why use Kubernetes?

เมื่อแอปพลิเคชันประกอบด้วยหลาย Containers การจัดการ Containers ด้วยตนเองอาจมีความซับซ้อน Kubernetes ช่วยให้เรากำหนดสถานะที่ต้องการของระบบ และใช้ Controllers คอยตรวจสอบและปรับสถานะจริงให้สอดคล้องกับสถานะที่กำหนดไว้

ตัวอย่างเช่น หากต้องการให้แอปพลิเคชันทำงาน 3 Replicas เราสามารถกำหนดจำนวน Replicas ไว้ใน Configuration แล้วให้ Kubernetes จัดการสร้างและดูแล Pods ตามที่ต้องการ

### Key Features

- **Container Orchestration:** จัดการการทำงานของ Containerized Applications ภายใน Cluster
- **Self-Healing:** ตรวจสอบและสร้าง Workloads ทดแทนเมื่อ Pods ที่อยู่ภายใต้ Controller หายไปหรือไม่ทำงานตามที่ต้องการ
- **Scaling:** เพิ่มหรือลดจำนวน Replicas ของแอปพลิเคชัน
- **Service Discovery & Networking:** ช่วยให้ Workloads ติดต่อกันผ่าน Services และเครือข่ายของ Cluster
- **Declarative Configuration:** กำหนด Desired State ผ่าน YAML หรือ JSON Manifests
- **Rolling Updates:** อัปเดต Workloads อย่างเป็นขั้นตอนผ่าน Deployment Strategy ที่กำหนด

Kubernetes ไม่ได้จัดการทุกเรื่องให้โดยอัตโนมัติ เช่น การตั้งค่าเครื่อง Host การ Build Application Images และการออกแบบระบบสำรองข้อมูล ยังต้องอาศัยเครื่องมือและการตั้งค่าเพิ่มเติมตามความต้องการของระบบ

## Architecture & Workflow

### Main Components

#### 1. Cluster

Cluster คือกลุ่มของ Machines ที่ทำงานร่วมกันเพื่อรันและจัดการ Workloads โดยประกอบด้วย Control Plane และ Worker Nodes อย่างน้อยหนึ่ง Node สำหรับรัน Pods

#### 2. Control Plane

Control Plane ทำหน้าที่ควบคุมและจัดการสถานะของ Cluster

- **kube-apiserver:** จุดรับ Requests ผ่าน Kubernetes API
- **etcd:** Distributed Key-Value Store ที่เก็บข้อมูลสถานะและ Configuration ของ Cluster
- **kube-scheduler:** เลือก Node ที่เหมาะสมสำหรับ Pods ที่ยังไม่ได้รับการจัดตารางให้รัน
- **kube-controller-manager:** รัน Controllers ที่ตรวจสอบสถานะจริงและพยายามปรับให้สอดคล้องกับ Desired State

#### 3. Worker Node

Worker Node ทำหน้าที่รัน Application Workloads ภายใน Cluster

- **kubelet:** Agent ที่ทำงานบน Node และตรวจสอบให้ Pods ที่ได้รับมอบหมายทำงานตามที่กำหนด
- **Container Runtime:** ซอฟต์แวร์ที่ใช้รัน Containers เช่น containerd หรือ CRI-O
- **Network Components:** Components ที่รองรับการสื่อสารระหว่าง Pods และ Services โดยรายละเอียดขึ้นอยู่กับการติดตั้งและ Networking Plugins ของ Cluster

#### 4. Core Kubernetes Objects

- **Pod:** หน่วยพื้นฐานที่ Kubernetes ใช้จัดการ Workloads โดยหนึ่ง Pod สามารถมีหนึ่งหรือหลาย Containers ที่ทำงานร่วมกัน
- **Deployment:** ใช้ประกาศและจัดการ Stateless Application โดยดูแล ReplicaSets และการเปลี่ยนแปลงของ Pods
- **ReplicaSet:** ช่วยรักษาจำนวน Pods ให้ตรงกับจำนวน Replicas ที่กำหนด โดยทั่วไป Deployment จะเป็นผู้จัดการ ReplicaSet ให้อีกที
- **Service:** ให้ Network Endpoint ที่คงที่สำหรับเข้าถึง Pods ซึ่งอาจถูกสร้างใหม่หรือเปลี่ยนตำแหน่งได้
- **ConfigMap:** ใช้เก็บ Configuration ที่ไม่ใช่ข้อมูลลับ
- **Secret:** ใช้จัดเก็บข้อมูลที่มีความอ่อนไหว เช่น Credentials หรือ Tokens โดยต้องตั้งค่าการรักษาความปลอดภัยเพิ่มเติมอย่างเหมาะสม

### How It Works

1. ผู้ใช้เขียน YAML Manifest เพื่อกำหนด Desired State ของ Resources
2. ใช้ `kubectl apply` ส่ง Configuration ไปยัง Kubernetes API Server
3. API Server ตรวจสอบ Request และบันทึกสถานะที่เกี่ยวข้องลงใน etcd
4. Controllers ตรวจพบ Resources ที่ต้องสร้างหรือปรับปรุง และดำเนินการให้สถานะเข้าใกล้ Desired State
5. Scheduler เลือก Node สำหรับ Pods ที่ยังไม่ได้รับการจัดตาราง
6. kubelet บน Node ที่ได้รับเลือกทำงานร่วมกับ Container Runtime เพื่อดึง Images หากจำเป็น และเริ่ม Containers
7. สถานะของ Resources ถูกอัปเดตกลับมายัง Kubernetes API ทำให้ระบบสามารถติดตามและปรับสถานะต่อเนื่องได้

### Architecture Overview

```text
             User / kubectl
                    |
                    v
              kube-apiserver
                    |
                    v
                  etcd
                    ^
                    |
         +----------+----------+
         |                     |
         v                     v
     Scheduler          Controllers
         |                     |
         +----------+----------+
                    |
             Assign Pod to Node
                    |
          +---------+---------+
          |                   |
          v                   v
      Worker Node         Worker Node
          |                   |
        kubelet             kubelet
          |                   |
    Container Runtime   Container Runtime
          |                   |
         Pods                 Pods
```

ภาพนี้เป็นแผนผังเชิงแนวคิดเพื่อแสดงองค์ประกอบหลัก ไม่ใช่ลำดับ Request ที่เกิดขึ้นทั้งหมดตามเวลาจริง เพราะ Controllers และ Components ต่าง ๆ ทำงานประสานกันอย่างต่อเนื่อง

## Configuration Reference

### 1. Kubernetes Manifest

Manifest คือไฟล์ YAML หรือ JSON ที่ใช้ประกาศ Configuration ของ Kubernetes Objects

ตัวอย่างโครงสร้างทั่วไป:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:stable
          ports:
            - containerPort: 80
```

องค์ประกอบสำคัญ:

- `apiVersion`: ระบุ Kubernetes API Version ที่ Resource ใช้
- `kind`: ระบุชนิดของ Resource
- `metadata`: ข้อมูลระบุ Resource เช่น ชื่อและ Namespace
- `spec`: Desired State ที่ต้องการให้ Resource มี
- `status`: สถานะปัจจุบันที่ Kubernetes รายงานกลับมา โดยระบบเป็นผู้ดูแลข้อมูลส่วนนี้เป็นหลัก

### 2. Deployment

Deployment ใช้จัดการการรัน Application Pods และการเปลี่ยนแปลงของ Workloads

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:stable
          ports:
            - containerPort: 80
```

- `replicas`: จำนวน Pod Replicas ที่ต้องการ
- `selector`: เงื่อนไขสำหรับเลือก Pods ที่ Deployment จัดการผ่าน ReplicaSet
- `template`: Template ที่ใช้สร้าง Pods
- `containers`: รายการ Containers ภายใน Pod
- `image`: Container Image ที่ต้องการใช้
- `containerPort`: เอกสารอธิบายพอร์ตที่ Container ใช้งาน ไม่ได้เปิดพอร์ตให้เข้าถึงจากภายนอกโดยอัตโนมัติ

**Important:** Labels ใน `spec.template.metadata.labels` ต้องสอดคล้องกับ Selector ของ Deployment

### 3. Service

Service ใช้กำหนด Endpoint สำหรับเข้าถึง Pods โดยอาศัย Labels และ Selectors

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

- `selector`: เลือก Pods ที่มี Label ตรงกัน
- `port`: พอร์ตที่ Service เปิดให้ใช้ภายใน Service Endpoint
- `targetPort`: พอร์ตปลายทางของ Application Pod
- `type: ClusterIP`: เปิด Service ให้เข้าถึงได้ภายใน Cluster เป็นหลัก

Service ประเภทอื่น เช่น `NodePort` และ `LoadBalancer` ใช้สำหรับรูปแบบการเข้าถึงที่แตกต่างกัน ส่วนการเข้าถึงจากภายนอกอาจต้องตั้งค่า Ingress, Gateway หรือระบบเครือข่ายเพิ่มเติม

### 4. Apply Configuration

สมมติว่าไฟล์ Deployment และ Service ถูกบันทึกไว้ใน Directory ชื่อ `k8s/`

```bash
kubectl apply -f k8s/
```

คำสั่งนี้ใช้สร้างหรือปรับปรุง Resources ให้ตรงตาม Manifests ที่กำหนด โดยต้องมี Cluster ที่ใช้งานได้และ `kubectl` ตั้งค่า Context ถูกต้องแล้ว

### 5. Scale and Update

ปรับจำนวน Replicas:

```bash
kubectl scale deployment/nginx --replicas=3
```

เปลี่ยน Image ของ Deployment:

```bash
kubectl set image deployment/nginx nginx=nginx:1.28
```

ตัวอย่าง Image ด้านบนใช้แสดง Syntax สำหรับอัปเดตเท่านั้น ควรเลือก Image Tag ที่มีอยู่จริงและเหมาะกับ Environment ก่อนใช้งาน

ตรวจสอบสถานะการอัปเดต:

```bash
kubectl rollout status deployment/nginx
```

การเปลี่ยนแปลงที่ทำผ่านคำสั่งเหล่านี้อาจไม่ได้ถูกบันทึกกลับไปยัง YAML ต้นฉบับ จึงควรปรับ Manifest ให้ตรงกับ Desired State ที่ต้องการเก็บไว้ด้วย

### 6. Namespace

Namespace ใช้แบ่งกลุ่ม Resources ภายใน Cluster เพื่อช่วยจัดการและแยกขอบเขตการทำงาน

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: development
```

เมื่อต้องการสร้าง Resource ภายใน Namespace สามารถระบุใน `metadata` ได้:

```yaml
metadata:
  name: nginx
  namespace: development
```

Namespace ไม่ใช่ขอบเขตการรักษาความปลอดภัยที่สมบูรณ์ในตัวเอง ต้องใช้ RBAC, Network Policies และการตั้งค่าอื่นตามความจำเป็น

## Common Workflow

### 1. Check the Environment

ตรวจสอบ Kubernetes CLI และ Cluster ที่เชื่อมต่ออยู่:

```bash
kubectl version
kubectl config current-context
kubectl get nodes
```

`kubectl` เป็น CLI สำหรับติดต่อ Cluster ไม่ใช่เครื่องมือที่สร้าง Kubernetes Cluster ให้โดยตัวมันเอง การสร้าง Cluster ต้องใช้วิธีที่เหมาะสม เช่น Minikube, kind หรือ Managed Kubernetes Service

### 2. Deploy an Application

```bash
kubectl apply -f k8s/
```

นำ Manifests ไปสร้างหรือปรับปรุง Resources ใน Cluster

### 3. Inspect Resources

```bash
kubectl get pods
kubectl get deployments
kubectl get services
kubectl get all
```

คำสั่งเหล่านี้ใช้ดู Resources ประเภทต่าง ๆ โดย `kubectl get all` ไม่ได้แสดง Resources ทุกชนิดใน Cluster

### 4. Debug an Application

ดูรายละเอียดของ Pod:

```bash
kubectl describe pod <pod-name>
```

ดู Logs ของ Container:

```bash
kubectl logs <pod-name>
```

ติดตาม Logs แบบต่อเนื่อง:

```bash
kubectl logs -f <pod-name>
```

หาก Pod มีหลาย Containers อาจต้องระบุชื่อ Container ด้วย `-c <container-name>`

### 5. Access an Application

สำหรับการทดสอบในเครื่อง สามารถ Forward Port จาก Service มายัง Host ได้:

```bash
kubectl port-forward service/nginx 8080:80
```

จากนั้นเข้าถึง `http://localhost:8080` ระหว่างที่คำสั่ง Port Forward ยังทำงานอยู่

### 6. Update and Roll Back

ตรวจสอบสถานะการ Deploy:

```bash
kubectl rollout status deployment/nginx
```

ดูประวัติ Revision:

```bash
kubectl rollout history deployment/nginx
```

ย้อนกลับ Revision ก่อนหน้า:

```bash
kubectl rollout undo deployment/nginx
```

การ Roll Back ใช้ประวัติของ Deployment ที่ยังมีอยู่ ไม่ใช่การย้อนกลับข้อมูลในฐานข้อมูลหรือข้อมูลที่ Application จัดเก็บไว้โดยอัตโนมัติ

### 7. Delete Resources

ลบ Resources ที่สร้างจากไฟล์ Manifest:

```bash
kubectl delete -f k8s/
```

หรือลบ Deployment โดยตรง:

```bash
kubectl delete deployment nginx
```

ตรวจสอบให้แน่ใจก่อนลบ Resources โดยเฉพาะใน Cluster ที่ใช้งานร่วมกับผู้อื่นหรือเป็นระบบ Production

### Important Notes

- Kubernetes จัดการ Desired State ของ Workloads ไม่ใช่เพียงสั่งให้ Containers เริ่มทำงานครั้งเดียว
- Pod สามารถถูกสร้างใหม่ได้ จึงไม่ควรพึ่งพาชื่อ Pod หรือ IP Address ของ Pod เป็น Endpoint ถาวร
- Deployment เหมาะสำหรับการจัดการ Stateless Workloads ส่วน Stateful Workloads อาจต้องใช้ StatefulSet และ Persistent Volumes
- `kubectl apply` ช่วยจัดการ Resources จากไฟล์ Manifest แต่การมี YAML เพียงอย่างเดียวไม่ได้รับประกันว่าจะสร้าง Environment ได้เหมือนกันทุกระบบ
- ความสามารถของ Cluster ขึ้นอยู่กับ Provider, Container Runtime, Networking Plugin, Storage Configuration และเวอร์ชันที่ใช้งาน
- ควรแยก Configuration สำหรับ Development, Testing และ Production ให้เหมาะสม รวมถึงกำหนดการจัดการ Secrets และสิทธิ์การเข้าถึงให้ปลอดภัย
