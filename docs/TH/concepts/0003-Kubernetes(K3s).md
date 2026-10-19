# K3s

## Overview

K3s เป็น Kubernetes Distribution ที่ออกแบบมาให้มีน้ำหนักเบา ติดตั้งและดูแลรักษาได้ง่าย โดยยังคงความเข้ากันได้กับ Kubernetes API เหมาะสำหรับการใช้งานบนเครื่องจริง Virtual Machines, Edge Computing, Home Lab, CI/CD และ Development Environment

K3s รวมองค์ประกอบที่จำเป็นสำหรับ Kubernetes Cluster ไว้ในแพ็กเกจที่ติดตั้งได้ง่าย ลด Dependencies และ Configuration ที่ผู้ดูแลต้องจัดการด้วยตนเอง

### Why use K3s?

การติดตั้ง Kubernetes แบบทั่วไปอาจต้องจัดการ Components, Certificates, Networking และ Datastore หลายส่วนด้วยตนเอง K3s ช่วยลดความซับซ้อนเหล่านี้ โดยรวบรวม Components หลักและค่าเริ่มต้นที่จำเป็นไว้ใน Distribution เดียว

ตัวอย่างเช่น สามารถติดตั้ง K3s บน Linux เครื่องเดียวเพื่อสร้าง Kubernetes Cluster ที่ทำงานได้จริง หรือเพิ่มเครื่องอื่นเข้ามาเป็น Agent Nodes เพื่อเพิ่มความสามารถในการรองรับ Workloads

### Key Features

- **Lightweight Distribution:** ลดความซับซ้อนในการติดตั้งและการจัดการ Kubernetes
- **Simple Installation:** มี Installation Script สำหรับติดตั้งและตั้งค่าเป็น System Service
- **Single-Node Cluster:** Server เครื่องเดียวสามารถทำหน้าที่เป็น Kubernetes Cluster ที่สมบูรณ์ได้
- **Server and Agent Nodes:** รองรับการแยกเครื่องที่ดูแล Control Plane ออกจากเครื่องที่ใช้รัน Workloads
- **Built-in Components:** รวม Components ที่ใช้บ่อย เช่น containerd, Flannel, CoreDNS, Traefik และ Local Path Provisioner
- **Flexible Datastore:** ใช้ SQLite เป็นค่าเริ่มต้นสำหรับ Single-Server Setup และรองรับ Embedded etcd รวมถึง External Databases
- **Kubernetes Compatibility:** รองรับ Kubernetes API และเครื่องมือมาตรฐาน เช่น `kubectl` และ YAML Manifests

K3s ไม่ใช่ Kubernetes API คนละชุด แต่เป็น Kubernetes Distribution ที่จัดแพ็กเกจและค่าเริ่มต้นให้เหมาะกับการติดตั้งและใช้งานที่เรียบง่ายขึ้น

## Architecture & Workflow

### Main Components

#### 1. K3s Server

Server Node เป็นเครื่องที่รัน `k3s server` และรับผิดชอบ Components หลักของ Control Plane รวมถึง Datastore ตามรูปแบบที่เลือกใช้

หน้าที่สำคัญ ได้แก่:

- เปิด Kubernetes API Server ให้ Clients และ Nodes ติดต่อ
- จัดการสถานะของ Cluster ผ่าน Datastore
- รัน Scheduler และ Controllers
- จัดการการลงทะเบียน Nodes และ Configuration ที่เกี่ยวข้อง

K3s Server ยังมี Kubelet, Container Runtime และ Networking Components สำหรับรัน Workloads ได้ด้วย ดังนั้น Server Node ไม่ได้จำเป็นต้องทำหน้าที่เฉพาะ Control Plane เสมอไป

#### 2. K3s Agent

Agent Node เป็นเครื่องที่รัน `k3s agent` เพื่อเข้าร่วม Cluster ที่มี Server อยู่แล้ว

หน้าที่สำคัญ ได้แก่:

- ลงทะเบียน Node เข้ากับ Cluster
- ทำงานร่วมกับ Kubelet เพื่อดูแล Pods ที่ได้รับมอบหมาย
- ใช้ Container Runtime ในการรัน Containers
- เชื่อมต่อกับ Kubernetes API ผ่าน Server

Agent ไม่ได้เป็นผู้ดูแล Control Plane หรือ Datastore ของ Cluster

#### 3. Datastore

Datastore ใช้จัดเก็บสถานะและข้อมูลของ Kubernetes Cluster

K3s รองรับรูปแบบหลัก ๆ ดังนี้:

- **SQLite:** ค่าเริ่มต้นสำหรับ Single-Server Setup
- **Embedded etcd:** ใช้ทำ High Availability (HA) Control Plane โดยทั่วไปต้องมี Server Nodes อย่างน้อย 3 เครื่อง
- **External Datastore:** เช่น MySQL, PostgreSQL หรือ etcd ภายนอก สำหรับ Cluster ที่ต้องการแยก Datastore ออกจาก Server

การเลือก Datastore ส่งผลต่อการออกแบบ High Availability, Backup และ Recovery

#### 4. Built-in Components

K3s รวม Components ที่จำเป็นหรือพบได้บ่อยหลายรายการไว้ให้แล้ว เช่น:

- **containerd:** Container Runtime ค่าเริ่มต้น
- **Flannel:** Container Network Interface (CNI) สำหรับเครือข่ายระหว่าง Pods
- **CoreDNS:** ให้บริการ DNS ภายใน Cluster
- **Traefik:** Ingress Controller ที่ติดตั้งมาโดยค่าเริ่มต้น
- **ServiceLB:** ช่วยให้ใช้งาน Services ประเภท LoadBalancer ได้โดยไม่จำเป็นต้องติดตั้ง Load Balancer Controller ภายนอกในกรณีพื้นฐาน
- **Local Path Provisioner:** ช่วยสร้าง Persistent Volumes โดยใช้ Storage ภายใน Node

Components บางรายการสามารถปิดหรือปรับเปลี่ยนได้ตามการออกแบบ Cluster

### How It Works

1. ติดตั้ง K3s บนเครื่อง Linux เพื่อสร้าง Server Node
2. K3s เริ่ม Components ของ Control Plane, Datastore และระบบที่จำเป็น
3. ผู้ใช้ส่ง Kubernetes Manifests ผ่าน `kubectl` ไปยัง Kubernetes API Server
4. Controllers ตรวจสอบ Desired State และสร้างหรือปรับปรุง Resources ที่เกี่ยวข้อง
5. Scheduler เลือก Node สำหรับ Pods ที่ต้องรัน
6. Kubelet และ Container Runtime บน Node ทำงานร่วมกันเพื่อเริ่ม Containers
7. เมื่อจำเป็น สามารถติดตั้ง K3s เพิ่มบนเครื่องอื่นและลงทะเบียนเป็น Agent Nodes ของ Cluster เดิมได้

### Architecture Overview

```text
                 User / kubectl
                       |
                       v
                K3s Server Node
                ┌──────────────┐
                │ API Server   │
                │ Scheduler    │
                │ Controllers  │
                │ Datastore    │
                └──────┬───────┘
                       |
                 Kubernetes API
                       |
             ┌─────────┴─────────┐
             |                   |
             v                   v
       K3s Agent Node       K3s Agent Node
       ┌────────────┐       ┌────────────┐
       │ Kubelet    │       │ Kubelet    │
       │ containerd │       │ containerd │
       │ Pods       │       │ Pods       │
       └────────────┘       └────────────┘
```

แผนผังนี้เป็นภาพรวมเชิงแนวคิด ใน Single-Node Setup องค์ประกอบของ Server และ Workloads สามารถอยู่บนเครื่องเดียวกันได้

## Configuration Reference

### 1. Install K3s

บน Linux เครื่องที่ใช้เป็น Server สามารถติดตั้ง K3s ด้วย Installation Script:

```bash
curl -sfL https://get.k3s.io | sh -
```

คำสั่งนี้จะติดตั้ง K3s และตั้งค่าให้ทำงานเป็น System Service โดยทั่วไปชื่อ Service คือ `k3s`

ควรอ่านและตรวจสอบ Installation Script ก่อนเรียกใช้งาน โดยเฉพาะบน Production Machines และควรเลือกเวอร์ชันที่จะติดตั้งให้ชัดเจนเมื่อต้องการควบคุมเวอร์ชันของ Cluster

### 2. K3s Configuration File

K3s อ่าน Configuration จากไฟล์ค่าเริ่มต้น:

```text
/etc/rancher/k3s/config.yaml
```

ตัวอย่าง Configuration สำหรับ Server:

```yaml
node-name: "k3s-server"

# Optional: เพิ่ม DNS name หรือ IP ที่ใช้เข้าถึง API Server
# tls-san:
#   - "k3s.example.local"
```

- `node-name`: กำหนดชื่อ Node
- `tls-san`: เพิ่ม DNS names หรือ IP addresses ที่ต้องการให้ใบรับรอง TLS ของ Server รองรับ

หลังจากแก้ Configuration อาจต้อง Restart Service เพื่อให้การตั้งค่ามีผล:

```bash
sudo systemctl restart k3s
```

ควรตรวจสอบผลกระทบของแต่ละ Option ก่อนแก้ไข โดยเฉพาะ Configuration ที่เกี่ยวข้องกับ Networking, Datastore และการเข้าร่วม Cluster

### 3. Join an Agent Node

บน Server สามารถอ่าน Token สำหรับลงทะเบียน Node ได้จาก:

```text
/var/lib/rancher/k3s/server/node-token
```

จากนั้นติดตั้ง K3s บนเครื่อง Agent โดยกำหนด URL ของ Server และ Token:

```bash
curl -sfL https://get.k3s.io | \
  K3S_URL=https://<server-ip>:6443 \
  K3S_TOKEN=<node-token> sh -
```

- `K3S_URL`: URL ของ K3s Server ที่ Agent ใช้เชื่อมต่อ
- `K3S_TOKEN`: Token ที่ใช้ยืนยันสิทธิ์ในการเข้าร่วม Cluster
- `6443`: พอร์ต Kubernetes API ที่ใช้ในตัวอย่างนี้

ต้องตรวจสอบว่าเครือข่าย Firewall และ Hostname ของแต่ละ Node ถูกตั้งค่าอย่างเหมาะสม และห้ามเปิดเผย Node Token ต่อบุคคลที่ไม่เกี่ยวข้อง

### 4. kubectl and Kubeconfig

K3s สร้าง Kubeconfig สำหรับผู้ดูแลไว้ที่:

```text
/etc/rancher/k3s/k3s.yaml
```

ตรวจสอบ Nodes:

```bash
sudo k3s kubectl get nodes
```

ดู Pods ทั้งหมดในทุก Namespaces:

```bash
sudo k3s kubectl get pods -A
```

Kubeconfig นี้มีสิทธิ์เข้าถึง Cluster ระดับสูง จึงควรรักษาสิทธิ์การอ่านไฟล์ไว้ให้เหมาะสม หากต้องการใช้งานจากเครื่องอื่น ต้องพิจารณา Server Address, TLS Certificate และสิทธิ์ของผู้ใช้ให้ถูกต้องด้วย

### 5. Manage K3s Components

K3s ติดตั้ง Components หลายรายการมาโดยค่าเริ่มต้น เช่น Traefik และ ServiceLB จึงควรตรวจสอบว่ามี Components ใดทำงานอยู่ก่อนเพิ่มระบบอื่นที่มีหน้าที่ซ้ำกัน

ตัวอย่างคำสั่งปิด Traefik ระหว่างติดตั้ง:

```bash
curl -sfL https://get.k3s.io | \
  sh -s - server --disable=traefik
```

หากต้องการใช้คำสั่งลักษณะนี้กับ Cluster ที่มีอยู่แล้ว ต้องวางแผน Configuration และผลกระทบต่อ Workloads ก่อน การปิด Components ไม่ใช่เพียงการลบ Pod ด้วย `kubectl delete`

### 6. Persistence and Storage

K3s มี Local Path Provisioner ให้ใช้งานเป็นค่าเริ่มต้น ซึ่งช่วยจัดสรร Persistent Volume จาก Storage ภายใน Node

อย่างไรก็ตาม Local Storage ผูกกับเครื่องที่เก็บข้อมูลอยู่ จึงต้องวางแผน Backup, Recovery และการย้าย Workloads หากต้องการให้ข้อมูลคงอยู่แม้ Node จะเสียหาย

สำหรับ Workloads ที่มีข้อมูลสำคัญ ควรเลือก Storage Solution ให้เหมาะกับรูปแบบการใช้งานและ Availability Requirements

## Common Workflow

### 1. Check Installation

```bash
sudo systemctl status k3s
```

ตรวจสอบ Log ของ Server:

```bash
sudo journalctl -u k3s -f
```

คำสั่ง `-f` ใช้ติดตาม Log แบบต่อเนื่อง

### 2. Check Cluster Status

```bash
sudo k3s kubectl get nodes -o wide
sudo k3s kubectl get pods -A
```

ใช้ตรวจสอบสถานะ Nodes, IP addresses และ Pods ภายใน Cluster

### 3. Deploy an Application

สร้าง Deployment จากไฟล์ Manifest:

```bash
sudo k3s kubectl apply -f deployment.yaml
```

ตรวจสอบผลลัพธ์:

```bash
sudo k3s kubectl rollout status deployment/<deployment-name>
sudo k3s kubectl get pods
```

### 4. Join Additional Nodes

ติดตั้ง K3s บนเครื่องใหม่ด้วย `K3S_URL` และ `K3S_TOKEN` จากนั้นกลับไปตรวจสอบที่ Server:

```bash
sudo k3s kubectl get nodes
```

หาก Node ยังไม่ปรากฏ ควรตรวจสอบ Service Status, Agent Logs, Token, Server Address และ Firewall

### 5. Debug Workloads

ตรวจสอบรายละเอียดของ Pod:

```bash
sudo k3s kubectl describe pod <pod-name>
```

อ่าน Logs:

```bash
sudo k3s kubectl logs <pod-name>
```

สำหรับ Pod ที่มีหลาย Containers สามารถระบุ Container ด้วย `-c <container-name>`

### 6. Stop or Restart K3s

บน Server:

```bash
sudo systemctl stop k3s
sudo systemctl start k3s
```

บน Agent:

```bash
sudo systemctl stop k3s-agent
sudo systemctl start k3s-agent
```

การหยุด Service บน Node อาจส่งผลต่อ Workloads และการสื่อสารใน Cluster จึงควรพิจารณาผลกระทบก่อนทำบนระบบที่ให้บริการอยู่

### 7. Uninstall K3s

บน Server:

```bash
sudo /usr/local/bin/k3s-uninstall.sh
```

บน Agent:

```bash
sudo /usr/local/bin/k3s-agent-uninstall.sh
```

การ Uninstall อาจลบ Local Cluster Datastore, Configuration และ Local Persistent Volume Data ควรสำรองข้อมูลที่จำเป็นและตรวจสอบผลกระทบก่อนดำเนินการ

### Important Notes

- K3s เป็น Kubernetes Distribution ไม่ใช่ Container Runtime และไม่ใช่ Container Image Builder
- Server Node สามารถรัน Workloads ได้ด้วย จึงไม่ควรสมมติว่า Pods จะอยู่บน Agent เท่านั้น
- Single-Server Setup ที่ใช้ SQLite ไม่ได้ให้ High Availability ของ Control Plane โดยอัตโนมัติ
- Agent ต้องเชื่อมต่อกับ Server ผ่าน Network และ Token ที่ถูกต้อง
- K3s ติดตั้ง Networking, Ingress และ Storage Components บางส่วนไว้ให้แล้ว ควรตรวจสอบค่าเริ่มต้นก่อนติดตั้ง Components ที่ทำหน้าที่ซ้ำกัน
- Kubeconfig และ Node Token เป็นข้อมูลสำคัญด้านความปลอดภัย ควรเก็บเป็นความลับและจำกัดสิทธิ์การเข้าถึง
- สำหรับ Production ควรออกแบบ Backup, Upgrade, High Availability, Security และ Monitoring ให้เหมาะสมกับระบบจริง
