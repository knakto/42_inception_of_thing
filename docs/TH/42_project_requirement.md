# 42 Inception of Things — Project Requirements

> Checklist อ้างอิงจากโจทย์ *Inception of Things* เวอร์ชัน 4.1 โดยแยกข้อบังคับออกจากคำแนะนำ

## ข้อกำหนดทั่วไป

- [x] ทำโปรเจกต์ทั้งหมดภายใน Virtual Machine
- [x] เก็บไฟล์โปรเจกต์และ configuration ไว้ใน repository ตามโครงสร้างที่กำหนด
- [x] ส่งส่วนบังคับไว้ในโฟลเดอร์ `p1/`, `p2/` และ `p3/` ที่ root ของ repository
- [x] เก็บสคริปต์ที่จำเป็นไว้ในโฟลเดอร์ `scripts/` และไฟล์ configuration ไว้ใน `confs/` ตามความเหมาะสม
- [x] ทำ Part 1, Part 2 และ Part 3 ตามลำดับ
- [x] ตรวจสอบให้แน่ใจว่าโปรเจกต์ทำงานได้บนเครื่องที่ใช้ประเมิน โดยใช้ไฟล์ที่อยู่ใน repository

## Part 1 — K3s และ Vagrant

- [x] สร้าง Virtual Machine 2 เครื่องด้วย Vagrant
- [x] ใช้ Linux distribution เวอร์ชัน stable ล่าสุดที่เลือก
- [x] ตั้งชื่อเครื่องและ hostname โดยใช้ login ของสมาชิกในทีมตามด้วย `S` สำหรับ Server และ `SW` สำหรับ ServerWorker
- [x] กำหนด IP `192.168.56.110` ให้ Server และ `192.168.56.111` ให้ ServerWorker
- [x] ใช้ network interface แยกสำหรับ IP ที่กำหนด
- [x] สามารถ SSH เข้า Virtual Machine ทั้งสองเครื่องได้โดยไม่ต้องใช้รหัสผ่าน
- [x] ติดตั้ง K3s แบบ Server/controller บนเครื่องแรก
- [x] ติดตั้ง K3s แบบ Agent บนเครื่องที่สอง และให้เข้าร่วม cluster ของ Server
- [x] ติดตั้งและใช้งาน `kubectl` สำหรับจัดการ Kubernetes
- [x] ระบุแหล่งที่มาของ VM box ใน Vagrant configuration ด้วย `config.vm.box_url` ที่ชี้ไปยัง image ของ distribution หรือใช้ box ที่สร้างขึ้นเอง

## Part 2 — K3s และเว็บแอปพลิเคชัน 3 ตัว

- [x] เตรียม Virtual Machine 1 เครื่อง โดยใช้ Linux distribution เวอร์ชัน stable ล่าสุดที่เลือก
- [x] ติดตั้ง K3s ในโหมด Server บนเครื่องดังกล่าว
- [x] Deploy เว็บแอปพลิเคชัน 3 ตัวที่เลือกลงใน K3s
- [x] เข้าถึงแอปพลิเคชันผ่าน IP `192.168.56.110` โดยเลือกแอปตาม HTTP `Host` header
- [x] เมื่อใช้ `Host: app1.com` ต้องเข้าถึง Application 1
- [x] เมื่อใช้ `Host: app2.com` ต้องเข้าถึง Application 2
- [x] เมื่อใช้ Host อื่น ต้องเลือก Application 3 เป็นค่าเริ่มต้น
- [x] กำหนดให้ Application 2 มี 3 replicas
- [x] เตรียมแสดง Ingress ระหว่างการประเมิน

## Part 3 — K3d และ Argo CD

- [x] ติดตั้ง K3d บน Virtual Machine
- [x] จัดเตรียม Docker และเครื่องมือ/แพ็กเกจที่จำเป็นสำหรับการใช้งาน K3d
- [x] มีสคริปต์สำหรับติดตั้งเครื่องมือที่จำเป็น เพื่อให้ใช้เตรียมระบบระหว่างการประเมินได้
- [x] สร้าง Kubernetes namespace ชื่อ `argocd` สำหรับ Argo CD
- [x] สร้าง Kubernetes namespace ชื่อ `dev` สำหรับแอปพลิเคชัน
- [x] สร้าง public GitHub repository สำหรับ configuration โดยชื่อ repository ต้องมี login ของสมาชิกในทีมอย่างน้อยหนึ่งคน
- [x] ตั้งค่าให้ Argo CD deploy แอปพลิเคชันใน namespace `dev` โดยอัตโนมัติจาก public GitHub repository
- [x] เตรียมแอปพลิเคชัน 2 เวอร์ชันที่แตกต่างกัน
- [x] เลือกใช้แอปพลิเคชันตัวอย่าง `wil42/playground` หรือแอปพลิเคชันที่พัฒนาขึ้นเอง
- [x] หากใช้แอปพลิเคชันของตนเอง ให้ publish Docker image ใน public Docker Hub repository และกำหนด tag เป็น `v1` และ `v2`
- [x] หากใช้แอปพลิเคชันของตนเอง ต้องทำให้เวอร์ชัน `v1` และ `v2` แตกต่างกันอย่างเห็นได้ชัด
- [x] เปลี่ยนเวอร์ชันแอปพลิเคชันผ่าน public GitHub repository และตรวจสอบว่า Argo CD นำเวอร์ชันใหม่ขึ้นใช้งานได้จริง

## การส่งงานและการประเมิน

- [x] มีโฟลเดอร์ `p1/`, `p2/` และ `p3/` อยู่ที่ root ของ repository
- [x] เก็บไฟล์ที่จำเป็นต่อการสร้างและตรวจสอบระบบไว้ใน repository
- [x] ตรวจสอบชื่อโฟลเดอร์และไฟล์ให้ถูกต้องก่อนส่ง
- [x] ตรวจสอบว่าสามารถทำซ้ำขั้นตอนติดตั้งและแสดงผลการทำงานบนเครื่องประเมินได้

## Bonus — GitLab (ไม่บังคับ)

- [x] เก็บงาน Bonus ในโฟลเดอร์ `bonus/` ที่ root ของ repository
- [x] รัน GitLab ภายในเครื่อง (locally)
- [x] ตั้งค่า GitLab ให้ทำงานร่วมกับ cluster
- [x] สร้าง namespace ชื่อ `gitlab`
- [x] ทำให้ workflow ทั้งหมดของ Part 3 ทำงานได้โดยใช้ local GitLab แทน GitHub

## คำแนะนำจากโจทย์ (ไม่ใช่ข้อบังคับด้านทรัพยากร)

- แนะนำให้จัดสรรทรัพยากร VM เท่าที่จำเป็น โดยโจทย์ยกตัวอย่าง 1 CPU และ RAM 512 MB (หรือ 1024 MB)
- อ่านเอกสารของ K3s, K3d, Kubernetes และเครื่องมือที่เลือกใช้เพิ่มเติมตามความจำเป็น
