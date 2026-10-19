# Argo CD

## Overview

Argo CD เป็น Declarative GitOps Continuous Delivery (CD) Tool สำหรับ Kubernetes ที่ใช้ Git Repository เป็นแหล่งอ้างอิงหลักของ Desired State ของ Application และ Kubernetes Resources

Argo CD จะตรวจสอบ Configuration ที่อยู่ใน Git เปรียบเทียบกับสถานะจริงภายใน Kubernetes Cluster และแสดงความแตกต่างระหว่างสองสถานะ ผู้ใช้สามารถสั่ง Sync ด้วยตนเอง หรือกำหนดให้ Argo CD Sync โดยอัตโนมัติเมื่อพบการเปลี่ยนแปลงตามนโยบายที่ตั้งไว้

### Why use Argo CD?

ในการ Deploy Application แบบทั่วไป CI/CD Pipeline อาจต้องมี Credentials สำหรับติดต่อ Kubernetes Cluster และสั่ง Deploy โดยตรง ซึ่งต้องจัดการสิทธิ์การเข้าถึงและขั้นตอนการ Deploy ภายใน Pipeline

Argo CD ช่วยแยกหน้าที่เหล่านี้ โดยให้ CI รับผิดชอบการทดสอบ Build และ Push Container Images ส่วน Argo CD รับผิดชอบการนำ Configuration จาก Git ไปปรับใช้กับ Kubernetes Cluster

ตัวอย่างเช่น เมื่อมีการเปลี่ยน Image Tag ในไฟล์ Manifest และ Commit ขึ้น Git Argo CD สามารถตรวจพบการเปลี่ยนแปลงและ Deploy เวอร์ชันใหม่ได้ตาม Sync Policy ที่กำหนด

### Key Features

- **GitOps:** ใช้ Git Repository เป็นแหล่งอ้างอิง Desired State ของ Application
- **Continuous Reconciliation:** ตรวจสอบความแตกต่างระหว่าง Configuration ใน Git กับสถานะจริงใน Cluster
- **Automated Synchronization:** Sync Configuration โดยอัตโนมัติตามนโยบายที่กำหนด
- **Drift Detection:** ตรวจจับการเปลี่ยนแปลงของ Resources ที่ไม่ตรงกับ Configuration ใน Git
- **Application Health Monitoring:** แสดงสถานะการ Sync และสุขภาพของ Application
- **Multi-Cluster Management:** จัดการการ Deploy ไปยัง Kubernetes Clusters หลายชุด
- **Configuration Management Integration:** รองรับ Plain YAML, Kustomize, Helm และเครื่องมือจัดการ Configuration อื่นที่ตั้งค่าไว้
- **Access Control:** กำหนดสิทธิ์และขอบเขตของ Application ผ่าน RBAC และ AppProjects

Argo CD ไม่ได้ทำหน้าที่ Build Container Images หรือรัน Automated Tests ให้โดยอัตโนมัติ หน้าที่เหล่านี้มักเป็นความรับผิดชอบของ CI Tools เช่น GitHub Actions, GitLab CI หรือ Jenkins

## Architecture & Workflow

### Main Components

#### 1. Argo CD API Server

API Server เป็นจุดเชื่อมต่อสำหรับ Web UI, Argo CD CLI และระบบภายนอกที่ต้องการจัดการ Applications

หน้าที่สำคัญ ได้แก่:

- จัดการ Application และแสดงสถานะ
- รับคำสั่ง Sync และการดำเนินการที่เกี่ยวข้อง
- จัดการ Authentication และ RBAC
- จัดการข้อมูลการเชื่อมต่อ Git Repositories และ Kubernetes Clusters ตามสิทธิ์ที่กำหนด
- รับ Webhook Events เพื่อช่วยเร่งการตรวจสอบการเปลี่ยนแปลง

#### 2. Repository Server

Repository Server ทำหน้าที่อ่าน Configuration จาก Git Repository หรือแหล่งข้อมูลที่รองรับ และสร้าง Kubernetes Manifests จากไฟล์ที่กำหนด

หน้าที่สำคัญ ได้แก่:

- ดึงข้อมูล Repository และ Revision ที่ต้องการ
- อ่านไฟล์จาก Path ที่ระบุใน Application
- Render Helm Charts หรือประมวลผล Configuration ผ่าน Kustomize และเครื่องมือที่รองรับ
- ส่ง Rendered Manifests ให้ Components ที่เกี่ยวข้องนำไปเปรียบเทียบกับสถานะจริง

Repository Server มี Cache เพื่อช่วยลดการอ่านข้อมูลซ้ำจาก Repository

#### 3. Application Controller

Application Controller เป็น Kubernetes Controller ที่ติดตามสถานะของ Applications และเปรียบเทียบ Desired State กับ Live State

หน้าที่สำคัญ ได้แก่:

- ตรวจสอบความแตกต่างระหว่าง Git และ Cluster
- แสดงสถานะ `Synced` หรือ `OutOfSync`
- ประเมิน Health ของ Application
- ดำเนินการ Sync เมื่อได้รับคำสั่งหรือเมื่อนโยบาย Automated Sync อนุญาต
- ตรวจสอบและปรับสถานะของ Application อย่างต่อเนื่อง

#### 4. Application

Application เป็น Custom Resource ของ Argo CD ที่ใช้กำหนดว่า Configuration อยู่ที่ไหน และต้อง Deploy ไปยัง Kubernetes Cluster ใด

องค์ประกอบหลัก ได้แก่:

- **Source:** Repository, Revision และ Path หรือ Chart ที่ใช้สร้าง Resources
- **Destination:** Cluster และ Namespace ปลายทาง
- **Project:** AppProject ที่ควบคุมขอบเขตการใช้งานของ Application
- **Sync Policy:** นโยบายสำหรับการ Sync และการจัดการ Resources

Application ไม่ใช่ตัว Application ที่กำลังรันอยู่ แต่เป็น Resource ที่ Argo CD ใช้ประกาศและติดตามการ Deploy Application

#### 5. AppProject

AppProject ใช้จัดกลุ่ม Applications และกำหนดขอบเขตที่แต่ละ Application สามารถใช้งานได้

ตัวอย่างข้อกำหนดที่ AppProject ควบคุมได้:

- อนุญาต Git Repositories ใดบ้าง
- Deploy ไปยัง Clusters และ Namespaces ใดได้
- อนุญาตให้สร้าง Kubernetes Resource Types ใด
- กำหนดบทบาทและสิทธิ์สำหรับการจัดการ Applications

ทุก Application ต้องอยู่ภายใต้ Project หนึ่ง โดย Project `default` ถูกสร้างไว้ให้แล้วสำหรับการเริ่มต้นใช้งาน แต่มีสิทธิ์ค่อนข้างกว้าง จึงควรสร้าง AppProjects ที่จำกัดสิทธิ์อย่างชัดเจนเมื่อใช้งานจริง

#### 6. Supporting Components

Argo CD มี Components อื่นที่ช่วยสนับสนุนการทำงาน เช่น:

- **Redis:** ใช้ Cache ข้อมูลเพื่อช่วยเพิ่มประสิทธิภาพ
- **Dex:** Identity Service ที่อาจใช้สำหรับ Single Sign-On ใน Configuration ที่เลือกใช้
- **ApplicationSet Controller:** ช่วยสร้างและจัดการ Applications จำนวนมากจาก Templates และแหล่งข้อมูลที่กำหนด
- **Notifications:** ส่งการแจ้งเตือนเกี่ยวกับเหตุการณ์หรือสถานะของ Applications เมื่อเปิดใช้และตั้งค่าไว้

Components บางรายการเป็น Optional หรือขึ้นอยู่กับรูปแบบการติดตั้ง

### How It Works

1. นักพัฒนาส่ง Code ไปยัง Git Repository และ CI Pipeline ทำการทดสอบ Build และ Push Container Image ตาม Workflow ของทีม
2. Configuration ของ Kubernetes เช่น Image Tag หรือ Helm Values ถูก Commit ไปยัง Repository ที่ Argo CD ติดตาม
3. Argo CD อ่าน Configuration จาก Git และสร้าง Kubernetes Manifests ที่ต้องการ
4. Application Controller เปรียบเทียบ Desired State กับ Live State ใน Cluster
5. หากพบความแตกต่าง Application จะแสดงสถานะ `OutOfSync`
6. ผู้ใช้สามารถสั่ง Sync เอง หรือให้ Argo CD Sync อัตโนมัติเมื่อเปิดใช้นโยบายดังกล่าว
7. Kubernetes Controllers จัดการ Workloads ตาม Resources ที่ได้รับ
8. Argo CD ตรวจสอบสถานะต่อไปเพื่อแสดง Sync Status และ Application Health

### Architecture Overview

```text
       Developer
           |
           v
       Git Repository <--------- CI Pipeline
           |                     Build / Test /
           |                     Push Image
           v
       Argo CD
       ├── API Server
       ├── Repository Server
       └── Application Controller
                   |
            Compare Desired State
              with Live State
                   |
                   v
            Kubernetes API Server
                   |
                   v
          Kubernetes Resources
          ├── Deployment
          ├── Service
          └── ConfigMap
```

Git เป็นแหล่งอ้างอิงของ Desired State ส่วน Application Controller ทำหน้าที่ Reconcile สถานะจริงใน Cluster ให้ตรงกับ Configuration ตามนโยบายที่กำหนด โดยไม่ได้หมายความว่าทุกการเปลี่ยนแปลงใน Git จะถูกนำไปใช้ทันที หากยังไม่ได้ Sync หรือยังไม่เปิด Automated Sync

### Important Status Concepts

- **Synced:** สถานะของ Resources ตรงกับ Desired State ที่ Argo CD คำนวณจาก Source
- **OutOfSync:** สถานะจริงแตกต่างจาก Desired State ที่กำหนด
- **Healthy:** Application อยู่ในสถานะสุขภาพที่ Argo CD ประเมินว่าปกติ
- **Progressing:** Resources บางส่วนกำลังเปลี่ยนสถานะหรือยังทำงานไม่เสร็จ
- **Degraded:** Resources หรือ Application มีสถานะสุขภาพผิดปกติตามเงื่อนไขที่ Argo CD ตรวจสอบได้

Sync Status และ Health เป็นคนละเรื่อง Application อาจ Synced แต่ยังไม่ Healthy ได้ เช่น Deployment มี Configuration ตรงกับ Git แต่ Pods ยังไม่พร้อมให้บริการ

## Configuration Reference

### 1. Install Argo CD

ตัวอย่างนี้ใช้สำหรับติดตั้ง Argo CD บน Kubernetes Cluster ที่ใช้งานได้ เช่น Cluster ที่สร้างด้วย k3d

สร้าง Namespace:

```bash
kubectl create namespace argocd
```

ติดตั้ง Components ด้วย Official Installation Manifest:

```bash
kubectl apply --server-side --force-conflicts \
  -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

ตรวจสอบสถานะ:

```bash
kubectl get pods -n argocd
```

ตัวอย่างนี้ใช้ Stable Installation Manifest เพื่อเริ่มต้นทดลองใช้งาน สำหรับ Production ควรเลือก Argo CD Version ที่ชัดเจนและทำตาม Installation และ Upgrade Guide ของเวอร์ชันนั้น

### 2. Access the Web UI

สำหรับ Local Development สามารถ Forward Port จาก Host ไปยัง Argo CD Server:

```bash
kubectl port-forward svc/argocd-server \
  -n argocd 8080:443
```

จากนั้นเปิด:

```text
https://localhost:8080
```

วิธีนี้ไม่จำเป็นต้องเปิด Argo CD ออกสู่เครือข่ายภายนอกโดยตรง

การติดตั้งแบบมาตรฐานอาจสร้าง Initial Admin Password ไว้ใน Secret สามารถอ่านได้ด้วย:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
echo
```

ชื่อผู้ใช้เริ่มต้นโดยทั่วไปคือ `admin` ควรเปลี่ยนรหัสผ่านและรักษา Credentials ให้ปลอดภัยหลังเริ่มใช้งาน ตามแนวทางการติดตั้งของเวอร์ชันที่ใช้อยู่

### 3. Application Manifest

Application สามารถประกาศผ่าน Kubernetes YAML ได้เช่นเดียวกับ Resources ประเภทอื่น

ตัวอย่าง:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: guestbook
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```

ความหมายของ Fields:

- `apiVersion`: API Version ของ Argo CD Application Resource
- `kind`: ระบุว่า Resource นี้เป็น `Application`
- `metadata.name`: ชื่อ Application ใน Argo CD
- `metadata.namespace`: Namespace ที่เก็บ Application Resource โดยทั่วไปคือ `argocd`
- `spec.project`: AppProject ที่ Application ใช้งาน
- `spec.source.repoURL`: Git Repository ที่เก็บ Application Configuration
- `spec.source.targetRevision`: Branch, Tag หรือ Commit ที่ต้องการติดตาม
- `spec.source.path`: Directory ใน Repository ที่มี Configuration ของ Application
- `spec.destination.server`: Kubernetes API Server ปลายทาง
- `spec.destination.namespace`: Namespace ที่ต้องการ Deploy
- `CreateNamespace=true`: อนุญาตให้ Argo CD สร้าง Destination Namespace หากยังไม่มี

ตัวอย่างนี้ใช้ Repository สาธารณะสำหรับการทดลอง และยังไม่ได้กำหนด Automated Sync ดังนั้นสามารถตรวจสอบและสั่ง Sync ด้วยตนเองก่อน

สร้าง Application:

```bash
kubectl apply -f application.yaml
```

### 4. Sync Policy

โดยทั่วไป Application จะใช้ Manual Sync จนกว่าจะกำหนด Automated Sync

หากต้องการให้ Argo CD Sync โดยอัตโนมัติเมื่อพบการเปลี่ยนแปลง สามารถเพิ่ม Configuration เช่น:

```yaml
spec:
  syncPolicy:
    automated:
      enabled: true
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

คุณสมบัติสำคัญ:

- `enabled: true`: เปิด Automated Sync
- `prune: true`: อนุญาตให้ลบ Resources ที่ Argo CD จัดการและไม่มีอยู่ใน Desired State อีกต่อไป
- `selfHeal: true`: อนุญาตให้ Argo CD แก้ไข Drift ที่เกิดจากการเปลี่ยนแปลงใน Cluster ซึ่งไม่ได้มาจาก Git
- `CreateNamespace=true`: สร้าง Destination Namespace หากยังไม่มี

**ข้อควรระวัง:** `prune: true` อาจทำให้ Resources ถูกลบเมื่อเอาออกจาก Git และ `selfHeal: true` อาจเขียนทับการแก้ไขที่ทำด้วยมือใน Cluster ควรเริ่มจาก Manual Sync ในระหว่างเรียนรู้ แล้วค่อยเปิด Automated Sync เมื่อเข้าใจผลกระทบแล้ว

### 5. Helm Integration

Argo CD สามารถใช้ Helm Charts เป็น Source ของ Application ได้

ตัวอย่าง Application ที่ใช้ Chart จาก Helm Repository:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://prometheus-community.github.io/helm-charts
    chart: prometheus
    targetRevision: "<chart-version>"
    helm:
      releaseName: prometheus
      values: |
        alertmanager:
          enabled: false
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```

`<chart-version>` เป็น Placeholder ต้องแทนด้วย Chart Version ที่มีอยู่จริงก่อนใช้งาน

เมื่อใช้ Helm ร่วมกับ Argo CD, Argo CD จะใช้ Helm ในการ Render Chart เป็น Kubernetes Manifests และเป็นผู้จัดการ Application Lifecycle ผ่านระบบของตนเอง ไม่ใช่การใช้ `helm install` แยกจาก Argo CD ตาม Workflow ปกติ

หาก Chart ต้องใช้ Values จำนวนมาก สามารถจัดเก็บ Values ไว้ใน Git หรืออ้างอิงไฟล์ Values ตามรูปแบบที่ Argo CD รองรับ

### 6. Multi-Environment Configuration

Application หลาย Environment สามารถใช้ Chart หรือ Manifest Template ชุดเดียวกัน แล้วแยกค่าตาม Environment ได้

ตัวอย่างโครงสร้าง Repository:

```text
gitops/
├── apps/
│   ├── dev/
│   │   └── values.yaml
│   ├── staging/
│   │   └── values.yaml
│   └── production/
│       └── values.yaml
└── charts/
    └── myapp/
        ├── Chart.yaml
        ├── values.yaml
        └── templates/
```

การแยก Environment อาจทำได้โดยใช้คนละ Directory, Helm Values, Kustomize Overlays หรือ Repository ตามความเหมาะสม

ควรแยกสิทธิ์และ Destination Cluster/Namespace ตามนโยบายของทีม โดยเฉพาะ Production Environment

## Common Workflow

### 1. Check Argo CD

ตรวจสอบ Pods:

```bash
kubectl get pods -n argocd
```

ตรวจสอบ Application Resources:

```bash
kubectl get applications -n argocd
```

### 2. Create an Application

นำ Application Manifest ที่กำหนด Source และ Destination ไว้ไปใช้:

```bash
kubectl apply -f application.yaml
```

ตรวจสอบว่า Application ถูกสร้างแล้ว:

```bash
kubectl get applications -n argocd
```

### 3. Inspect Application Status

หากติดตั้ง Argo CD CLI และ Login ไว้แล้ว สามารถตรวจสอบสถานะด้วย:

```bash
argocd app list
argocd app get guestbook
```

ดูความแตกต่างระหว่าง Desired State และ Live State:

```bash
argocd app diff guestbook
```

คำสั่งเหล่านี้ช่วยตรวจสอบว่า Application อยู่ในสถานะ Synced หรือ OutOfSync และมี Resources ใดต้องปรับปรุง

### 4. Sync an Application

สั่ง Sync ด้วยตนเอง:

```bash
argocd app sync guestbook
```

รอให้สถานะการ Sync เสร็จ:

```bash
argocd app wait guestbook
```

หาก Application ใช้ Automated Sync และนโยบายทำงานอยู่ การ Sync อาจเกิดขึ้นเองเมื่อ Argo CD ตรวจพบการเปลี่ยนแปลง

### 5. Update an Application

Workflow เมื่ออัปเดต Application:

1. Build และ Test Application ผ่าน CI
2. Push Image ไปยัง Container Registry
3. แก้ Image Tag หรือ Configuration ใน Git Repository ที่ Argo CD ติดตาม
4. Commit และ Push การเปลี่ยนแปลง
5. Argo CD ตรวจพบ Configuration ใหม่
6. Sync ตามนโยบายที่กำหนด
7. ตรวจสอบ Sync Status และ Application Health

การ Push Source Code หรือ Container Image เพียงอย่างเดียวไม่ได้ทำให้ Argo CD Deploy เวอร์ชันใหม่เสมอไป หาก Configuration ที่ Argo CD ติดตามยังไม่ได้เปลี่ยนแปลง ก็อาจไม่มีการ Deploy ใหม่เกิดขึ้น

### 6. Roll Back an Application

ใน Workflow แบบ GitOps โดยทั่วไปแนะนำให้แก้ไข Desired State ใน Git กลับไปยัง Configuration หรือ Commit ที่ต้องการ แล้วให้ Argo CD Sync ใหม่

ตัวอย่าง:

```bash
git revert <commit-sha>
git push
```

วิธีนี้ช่วยให้ Git ยังคงเป็นแหล่งอ้างอิงของ Desired State และตรวจสอบประวัติการเปลี่ยนแปลงได้

การ Rollback ผ่าน Argo CD โดยตรงอาจไม่เหมาะกับ Automated Sync บางรูปแบบ เพราะสถานะที่แตกต่างจาก Git อาจถูก Sync กลับมาอีกครั้ง

### 7. Delete an Application

ลบ Application Resource ได้ด้วย:

```bash
kubectl delete -f application.yaml
```

การลบ Application อาจส่งผลต่อ Kubernetes Resources ที่ Application จัดการอยู่ ขึ้นอยู่กับ Finalizers และนโยบายการลบที่กำหนด จึงควรตรวจสอบผลกระทบก่อนดำเนินการ โดยเฉพาะใน Production

### Important Notes

- Git เป็นแหล่ง Desired State ส่วน Kubernetes Cluster เก็บ Live State ที่กำลังทำงานจริง
- `OutOfSync` ไม่ได้หมายความว่า Application ล่มเสมอไป แต่หมายถึงสถานะจริงแตกต่างจาก Desired State
- Automated Sync, Pruning และ Self-Heal เป็นคนละพฤติกรรมและควรเข้าใจผลกระทบของแต่ละตัว
- Helm ที่ใช้ผ่าน Argo CD มี Lifecycle Management ต่างจากการติดตั้ง Chart ด้วย Helm CLI โดยตรง
- ผู้ที่มีสิทธิ์เขียนลง Git Repository ที่ Argo CD เชื่อถือ อาจมีอิทธิพลต่อ Resources ที่ถูก Deploy ได้ จึงต้องควบคุมสิทธิ์ Repository อย่างเหมาะสม
- ควรจำกัด AppProject, RBAC และ Destination Clusters ให้ตรงกับสิทธิ์ที่ Application จำเป็นต้องใช้
- GitOps ช่วยให้ตรวจสอบการเปลี่ยนแปลงได้ แต่ไม่ได้ทดแทนการออกแบบ Secrets Management, Testing, Monitoring และ Backup
