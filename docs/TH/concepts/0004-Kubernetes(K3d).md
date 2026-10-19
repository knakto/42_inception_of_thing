# k3d

## Overview

k3d เป็นเครื่องมือสำหรับสร้างและจัดการ K3s Clusters ภายใน Docker โดยแต่ละ Kubernetes Node จะทำงานอยู่ภายใน Docker Container แทนการใช้ Physical Machine หรือ Virtual Machine แยกกัน

k3d เหมาะสำหรับ Local Development, Testing, CI/CD และการทดลอง Kubernetes Architecture เพราะสามารถสร้าง Cluster ที่มีหลาย Nodes ได้อย่างรวดเร็ว และลบเพื่อเริ่มต้นใหม่ได้ง่าย

### Why use k3d?

การทดลอง Kubernetes อาจต้องใช้หลาย Nodes เพื่อทดสอบการกระจาย Workloads, Networking และการทำงานระหว่าง Components หากสร้าง VM แยกสำหรับทุก Node จะใช้ทรัพยากรและมีขั้นตอน Setup มากขึ้น

k3d ช่วยลดความซับซ้อนโดยใช้ Docker Containers แทนเครื่องแยก ทำให้สามารถจำลอง Cluster สำหรับการพัฒนาและทดสอบได้ภายในเครื่องเดียว

### Key Features

- **Lightweight Clusters:** สร้าง K3s Clusters โดยใช้ Docker Containers เป็น Nodes
- **Multi-Node Support:** กำหนดจำนวน Server และ Agent Nodes ได้
- **Fast Lifecycle Management:** สร้าง เริ่ม หยุด และลบ Cluster ได้ด้วย CLI
- **Port Mapping:** เปิดพอร์ตจาก Host ไปยัง Cluster ผ่าน Docker Port Mapping และ Load Balancer
- **Local Image Import:** นำ Container Images ที่ Build ไว้ในเครื่องเข้าสู่ Cluster ได้
- **Declarative Configuration:** กำหนดการสร้าง Cluster ผ่าน YAML Configuration File
- **Multiple Clusters:** สร้าง Cluster หลายชุดเพื่อแยก Environment สำหรับการทดลองแต่ละแบบได้

### Relationship Between Kubernetes, K3s, and k3d

- **Kubernetes:** ระบบสำหรับจัดการ Containerized Workloads
- **K3s:** Kubernetes Distribution ที่มีน้ำหนักเบาและติดตั้งง่าย
- **k3d:** เครื่องมือที่ใช้สร้างและจัดการ K3s Clusters ภายใน Docker
- **Docker:** ระบบ Container Runtime สำหรับรัน Node Containers ที่ k3d สร้างขึ้น

k3d ไม่ใช่ Kubernetes Distribution และไม่ได้เข้ามาแทนที่ K3s แต่เป็นเครื่องมือที่ช่วยให้สร้างและจัดการ K3s Cluster ได้สะดวกขึ้น

## Architecture & Workflow

### Main Components

#### 1. Host Machine

Host Machine คือเครื่องจริงหรือ VM ที่ใช้รัน Docker, k3d CLI และเครื่องมืออย่าง `kubectl`

ผู้ใช้เรียกคำสั่ง `k3d cluster create` จาก Host เพื่อสั่งให้ k3d สร้าง Cluster

#### 2. Docker Engine

Docker Engine เป็นส่วนที่สร้างและจัดการ Containers ซึ่งทำหน้าที่เป็น Kubernetes Nodes

k3d ต้องพึ่งพา Docker ที่ทำงานได้ จึงไม่สามารถสร้าง Cluster ได้หาก Docker Engine ใช้งานไม่ได้

#### 3. K3s Node Containers

k3d สร้าง Containers ที่รัน K3s โดยแบ่งเป็น:

- **Server Node:** รัน Control Plane Components และอาจรัน Application Workloads ได้ด้วย
- **Agent Node:** ใช้รัน Application Workloads ภายใต้การจัดการของ Cluster

แม้ Node จะเป็น Docker Container แต่ภายใน Node ยังมี Components ของ K3s เช่น kubelet และ containerd สำหรับจัดการ Kubernetes Workloads

#### 4. Load Balancer Container

โดยค่าเริ่มต้น k3d สร้าง Load Balancer Container ไว้ด้านหน้า Server Nodes เพื่อช่วยรับและส่งต่อการเชื่อมต่อไปยัง Server Nodes

- ใช้เป็นจุดเชื่อมต่อสำหรับ Kubernetes API
- รองรับการ Forward พอร์ตจาก Host เข้า Cluster เมื่อกำหนด Port Mapping
- ช่วยให้การเข้าถึง Server Nodes ไม่จำเป็นต้องผูกกับ Server Node ใด Node หนึ่งโดยตรง

Load Balancer นี้เป็นส่วนหนึ่งของโครงสร้างที่ k3d จัดการ และสามารถปิดการสร้างได้ด้วย Configuration ที่รองรับ

#### 5. Docker Network

k3d สร้าง Docker Network สำหรับ Cluster เพื่อให้ Node Containers ติดต่อกันได้

อย่างไรก็ตาม Docker Network สำหรับ Nodes กับ Kubernetes Pod Network เป็นคนละระดับของเครือข่าย การเข้าถึง Application จาก Host จึงอาจต้องกำหนด Port Mapping, Service หรือ Ingress เพิ่มเติม

### How It Works

1. ผู้ใช้สั่ง `k3d cluster create` พร้อมระบุชื่อ Cluster และจำนวน Nodes ที่ต้องการ
2. k3d ติดต่อ Docker Engine เพื่อสร้าง Node Containers, Docker Network และ Load Balancer ตาม Configuration
3. K3s เริ่มทำงานภายใน Node Containers และสร้าง Kubernetes Cluster
4. k3d จัดเตรียม Kubeconfig เพื่อให้ `kubectl` เชื่อมต่อ Cluster ได้
5. ผู้ใช้สร้าง Kubernetes Resources ด้วย YAML Manifests ตามปกติ
6. Kubernetes Scheduler และ Controllers จัดการ Workloads ภายใน Cluster
7. เมื่อทดลองเสร็จ ผู้ใช้สามารถหยุดหรือ ลบ Cluster ผ่าน k3d ได้

### Architecture Overview

```text
                  Host Machine
        ┌────────────────────────────┐
        │ k3d CLI       kubectl      │
        │              Docker Engine │
        └────────────────┬───────────┘
                         |
                   Docker Network
                         |
             ┌───────────┴───────────┐
             |                       |
             v                       v
      Load Balancer           K3s Server Node
      Container               Container
             |                ┌─────────────┐
             └───────────────>│ Control     │
                              │ Plane       │
                              │ kubelet     │
                              │ containerd  │
                              └──────┬──────┘
                                     |
                          ┌──────────┴──────────┐
                          |                     |
                          v                     v
                    Agent Node 0          Agent Node 1
                    Container             Container
                    ┌───────────┐         ┌───────────┐
                    │ kubelet   │         │ kubelet   │
                    │ containerd│         │ containerd│
                    │ Pods      │         │ Pods      │
                    └───────────┘         └───────────┘
```

แผนผังนี้เป็นตัวอย่าง Cluster ที่มี Server 1 Node และ Agent 2 Nodes โดยโครงสร้างจริงอาจแตกต่างกันตามจำนวน Nodes, Load Balancer และ Configuration ที่เลือกใช้

## Configuration Reference

### 1. Requirements

เครื่องที่ใช้ k3d ต้องมี:

- Docker Engine ที่ทำงานได้
- k3d CLI
- `kubectl` สำหรับติดต่อ Kubernetes API

ตรวจสอบเครื่องมือ:

```bash
docker version
k3d version
kubectl version --client
```

### 2. Create a Cluster

สร้าง Cluster ชื่อ `dev` ซึ่งมี Server 1 Node และ Agent 2 Nodes:

```bash
k3d cluster create dev \
  --servers 1 \
  --agents 2
```

- `dev`: ชื่อ Cluster
- `--servers 1`: จำนวน Server Nodes
- `--agents 2`: จำนวน Agent Nodes

คำสั่งนี้สร้าง Kubernetes Cluster สำหรับ Development ภายใน Docker โดย k3d ยังจัดการ Network และ Load Balancer ที่เกี่ยวข้องให้ด้วยตามค่าเริ่มต้น

### 3. Port Mapping

หากต้องการเข้าถึง Application ผ่าน Host ให้กำหนด Port Mapping ตอนสร้าง Cluster:

```bash
k3d cluster create dev \
  --servers 1 \
  --agents 2 \
  -p "8081:80@loadbalancer"
```

ความหมายของ `"8081:80@loadbalancer"`:

- `8081`: พอร์ตบน Host
- `80`: พอร์ตบน Load Balancer Container
- `@loadbalancer`: ระบุให้ Port Mapping นี้ใช้กับ Load Balancer ที่ k3d จัดการ

เมื่อมี Ingress Controller หรือการตั้งค่าที่เหมาะสมแล้ว จะสามารถเข้าถึงบริการผ่านพอร์ต Host ที่กำหนดได้

การ Map พอร์ตเพียงอย่างเดียวไม่ได้สร้าง Application, Kubernetes Service หรือ Ingress ให้โดยอัตโนมัติ

### 4. Cluster Configuration File

หากต้องการเก็บ Configuration ไว้ใน Git เพื่อสร้าง Cluster ซ้ำได้ ให้ใช้ YAML Configuration File ของ k3d

สร้างไฟล์ตัวอย่างด้วย:

```bash
k3d config init --output k3d.yaml
```

จากนั้นแก้ไขค่าในไฟล์ให้ตรงกับ Environment ที่ต้องการ

ตัวอย่าง Configuration รูปแบบหนึ่ง:

```yaml
apiVersion: k3d.io/v1alpha5
kind: Simple
metadata:
  name: dev

servers: 1
agents: 2

ports:
  - port: "8081:80"
    nodeFilters:
      - loadbalancer
```

สร้าง Cluster จากไฟล์:

```bash
k3d cluster create --config k3d.yaml
```

`apiVersion` ของ k3d Configuration File อาจเปลี่ยนไปตามเวอร์ชันของ k3d จึงควรใช้ไฟล์ที่สร้างจาก `k3d config init` ของเวอร์ชันที่ติดตั้ง หรืออ้างอิง Schema ในเอกสารทางการ

### 5. Kubeconfig

k3d สามารถจัดการ Kubeconfig เพื่อให้ `kubectl` เชื่อมต่อ Cluster ที่สร้างขึ้นได้

สร้าง Cluster ใหม่โดยปกติ k3d จะอัปเดต Default Kubeconfig และสลับ Context ไปยัง Cluster ใหม่ตามค่าเริ่มต้น

ตรวจสอบ Context ที่ใช้งาน:

```bash
kubectl config current-context
```

ตรวจสอบ Nodes:

```bash
kubectl get nodes
```

หากต้องการรับ Kubeconfig ของ Cluster ที่มีอยู่:

```bash
k3d kubeconfig get dev
```

Kubeconfig มีข้อมูลสำหรับเชื่อมต่อและยืนยันสิทธิ์กับ Cluster จึงควรเก็บรักษาอย่างเหมาะสม

### 6. Import Local Container Images

เมื่อ Build Image บน Host แล้ว ต้องทำให้ Image นั้นเข้าถึงได้จาก Container Runtime ภายใน k3d Nodes ด้วย

ตัวอย่าง:

```bash
docker build -t demo:v1 .
k3d image import demo:v1 --cluster dev
```

จากนั้นสามารถอ้างอิง Image ใน Kubernetes Manifest ได้:

```yaml
containers:
  - name: demo
    image: demo:v1
    imagePullPolicy: IfNotPresent
```

การ Import Image มีประโยชน์สำหรับ Local Development เพราะไม่ต้อง Push Image ไปยัง Remote Registry ทุกครั้ง ทั้งนี้ Image ต้องถูก Import เข้า Cluster ที่ใช้จริง และ Tag ใน Manifest ต้องตรงกับ Image ที่มีอยู่

### 7. Volumes and Persistent Data

k3d สามารถ Map Directory จาก Host เข้าไปยัง Node Containers ได้ เช่น:

```bash
k3d cluster create dev \
  --volume "$PWD/data:/mnt/data@all"
```

ตัวอย่างนี้ Map Directory `data` ของ Host เข้าไปยัง `/mnt/data` ใน Node Containers ที่ตรงกับ Node Filter `@all`

การ Map Volume เข้า Node ไม่ได้ทำให้ Kubernetes สร้าง PersistentVolume หรือ PersistentVolumeClaim ให้อัตโนมัติ ต้องตั้งค่า Kubernetes Storage Resources แยกต่างหาก

นอกจากนี้ ข้อมูลใน Container และ Storage ภายใน Node อาจสูญหายเมื่อ Cluster ถูกลบ หากไม่ได้จัดเตรียม Persistent Storage ที่เหมาะสม

## Common Workflow

### 1. Create the Cluster

```bash
k3d cluster create dev --servers 1 --agents 2
```

สร้าง Cluster สำหรับ Development

### 2. Verify the Cluster

```bash
k3d cluster list
kubectl get nodes -o wide
kubectl get pods -A
```

ตรวจสอบว่า Cluster ถูกสร้างเรียบร้อยและ Nodes พร้อมใช้งาน

### 3. Deploy an Application

นำ Kubernetes Manifest ไปใช้งาน:

```bash
kubectl apply -f k8s/
```

ตรวจสอบ Resources:

```bash
kubectl get deployments
kubectl get pods
kubectl get services
```

### 4. Access the Application

หากต้องการทดสอบ Service โดยไม่ต้องตั้งค่า Host Port Mapping เพิ่ม สามารถใช้:

```bash
kubectl port-forward service/<service-name> 8080:80
```

จากนั้นเข้าถึง `http://localhost:8080` ขณะที่คำสั่งยังทำงานอยู่

สำหรับการทดสอบ Ingress หรือการเข้าถึงผ่าน Host Port ต้องกำหนด Port Mapping ตอนสร้าง Cluster และตั้งค่า Ingress หรือ Service ให้เหมาะสมด้วย

### 5. Load a Local Image

```bash
docker build -t demo:v1 .
k3d image import demo:v1 --cluster dev
kubectl apply -f deployment.yaml
```

ใช้ Workflow นี้เมื่อกำลังพัฒนา Application และต้องการทดสอบ Image ที่ Build ในเครื่อง

### 6. Stop and Start the Cluster

หยุด Cluster:

```bash
k3d cluster stop dev
```

เริ่ม Cluster อีกครั้ง:

```bash
k3d cluster start dev
```

การหยุด Cluster จะหยุด Node Containers แต่ไม่ได้ลบ Cluster

### 7. Delete the Cluster

```bash
k3d cluster delete dev
```

ลบ Cluster และทรัพยากรที่ k3d จัดการสำหรับ Cluster นั้น ควรตรวจสอบและสำรองข้อมูลสำคัญก่อน เพราะข้อมูลที่อยู่ภายใน Containers หรือ Volumes บางประเภทอาจไม่สามารถกู้คืนได้หลังลบ

### Important Notes

- k3d ใช้ Docker เพื่อรัน K3s Nodes ไม่ใช่การจำลอง Nodes ด้วย Virtual Machines
- k3d เหมาะกับ Local Development และ Testing แต่ไม่ได้จำลองคุณสมบัติของ Physical Machines และ Production Networking ได้เหมือนทุกประการ
- การเปิดพอร์ตต้องพิจารณาทั้ง Docker Port Mapping, Kubernetes Service และ Ingress Configuration
- Local Container Images ต้องถูก Import หรือเผยแพร่ผ่าน Registry ที่ Cluster เข้าถึงได้
- การ Map Host Directory เข้า Node ไม่เท่ากับการสร้าง Kubernetes Persistent Storage
- การลบ Cluster อาจทำให้ข้อมูลภายใน Cluster สูญหาย ควรสำรองข้อมูลก่อนดำเนินการ
