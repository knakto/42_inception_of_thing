# Vagrant

## Overview

Vagrant คือเครื่องมือสำหรับสร้างและจัดการ Virtual Development Environment โดยใช้ไฟล์ `Vagrantfile` เพื่อกำหนด Configuration ของ Virtual Machine (VM) และทำงานร่วมกับ Provider เช่น VirtualBox หรือ VMware เพื่อสร้างและจัดการ VM

### Why use Vagrant?

Vagrant ช่วยให้เราสามารถกำหนด Environment เป็น Code ทำให้สามารถสร้าง Environment ที่มี Configuration เหมือนกันซ้ำได้บนเครื่องที่แตกต่างกัน ลดปัญหาความแตกต่างของสภาพแวดล้อมระหว่างนักพัฒนา เช่น "It works on my machine"

### Key Features

- **Reproducible Environment:** กำหนด Configuration เพื่อให้สามารถสร้าง Environment กลับมาได้อย่างสม่ำเสมอ
- **Multiple Virtual Machines:** กำหนดและจัดการ VM หลายเครื่องภายในโปรเจกต์เดียว
- **Provisioning:** ติดตั้งซอฟต์แวร์และกำหนดค่า System โดยอัตโนมัติผ่าน Shell Script หรือเครื่องมืออย่าง Ansible
- **Resource Management:** กำหนด Resource เช่น CPU และ Memory ผ่าน Provider ที่รองรับ
- **Networking:** กำหนด Network ระหว่าง VM และการ Forward Port ระหว่าง Host กับ Guest
- **SSH Access:** เชื่อมต่อเข้า VM ผ่าน SSH ได้สะดวกด้วย `vagrant ssh`

## Architecture & Workflow

### Main Components

- **Vagrantfile:** ไฟล์ที่ใช้กำหนด Configuration ของ VM และ Environment โดยใช้ Ruby Syntax
- **Box:** Template Package ที่ใช้เป็นต้นแบบสำหรับสร้าง VM ซึ่งอาจประกอบด้วย Operating System และ Configuration พื้นฐาน
- **Provider:** ซอฟต์แวร์หรือระบบที่ Vagrant ใช้ติดต่อเพื่อสร้างและจัดการ VM เช่น VirtualBox หรือ VMware
- **Virtual Machine (VM):** เครื่องเสมือนที่ถูกสร้างขึ้นและใช้เป็น Environment สำหรับการทำงาน
- **Provisioner:** เครื่องมือหรือ Script ที่ใช้ติดตั้งซอฟต์แวร์และกำหนดค่า Environment ภายใน VM

### How it works

1. ผู้ใช้กำหนด Configuration ใน `Vagrantfile`
2. Vagrant อ่าน Configuration และตรวจสอบ Box ที่ต้องใช้
3. Vagrant ติดต่อ Provider เพื่อสร้างหรือเริ่ม VM
4. Vagrant ตั้งค่า Network และ Resource ตาม Configuration ที่กำหนด
5. Vagrant เรียกใช้ Provisioner หากมีการกำหนดไว้
6. ผู้ใช้สามารถเข้าถึง VM ผ่าน SSH และเริ่มทำงานได้

### Workflow Overview

`Vagrantfile` → Vagrant → Provider → Virtual Machine → Provisioning → Ready Environment

## Configuration Reference

### 1. Initialize a Project

สร้าง `Vagrantfile` เริ่มต้นโดยใช้คำสั่ง:

```bash
vagrant init bento/ubuntu-24.04
```

ตัวอย่าง Configuration พื้นฐาน:

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "bento/ubuntu-24.04"
end
```

- `Vagrant.configure("2")` กำหนด Configuration Schema Version
- `config.vm.box` กำหนด Box ที่ใช้สร้าง VM

หากต้องการกำหนดเวอร์ชันของ Box สามารถใช้ `config.vm.box_version` ได้ โดยต้องตรวจสอบว่าเวอร์ชันที่ระบุมีอยู่จริง

### 2. Provisioning

Provisioning ใช้สำหรับติดตั้งซอฟต์แวร์หรือกำหนดค่า Environment ภายใน VM โดยอัตโนมัติ

ตัวอย่างการใช้ Inline Shell:

```ruby
config.vm.provision "shell",
  name: "install-packages",
  inline: "sudo apt-get update"
```

ตัวอย่างการใช้ Shell Script:

```ruby
config.vm.provision "shell",
  name: "setup",
  path: "scripts/setup.sh"
```

- `name` ใช้ระบุชื่อ Provisioner
- `inline` ใช้ระบุคำสั่ง Shell โดยตรง
- `path` ใช้ระบุไฟล์ Script ที่ต้องการรัน

การ Provision สามารถสั่งทำงานอีกครั้งด้วย:

```bash
vagrant provision
```

หรือสั่ง Provision ระหว่างการเริ่ม VM:

```bash
vagrant up --provision
```

### 3. Synced Folders

Synced Folder ใช้สำหรับแชร์ไฟล์ระหว่าง Host Machine และ Guest VM

```ruby
config.vm.synced_folder "./project", "/home/vagrant/project",
  create: true
```

- Path แรกคือ Directory บน Host
- Path ที่สองคือ Directory ภายใน Guest
- `create: true` อนุญาตให้สร้าง Directory ปลายทางใน Guest หากยังไม่มี โดยขึ้นอยู่กับการรองรับของ Synced Folder Type

ควรพิจารณาเรื่อง File Permissions และประสิทธิภาพการเข้าถึงไฟล์ตาม Provider และระบบไฟล์ที่ใช้งาน

### 4. Hostname

กำหนด Hostname ของ VM:

```ruby
config.vm.hostname = "web-server"
```

Hostname ใช้ระบุชื่อเครื่องภายใน Environment แต่ไม่ได้ทำให้ VM สามารถติดต่อกันได้โดยอัตโนมัติ

### 5. Networking

#### Private Network

ใช้กำหนด IP Address สำหรับการสื่อสารผ่าน Private Network:

```ruby
config.vm.network "private_network",
  ip: "192.168.56.10"
```

ควรเลือก IP Address ที่ไม่ขัดแย้งกับ Network ที่ใช้งานอยู่ และตรวจสอบข้อกำหนดของ Provider

#### Forwarded Port

ใช้ส่งต่อพอร์ตจาก Host ไปยัง Guest:

```ruby
config.vm.network "forwarded_port",
  guest: 8080,
  host: 8080
```

ตัวอย่างนี้ทำให้สามารถเข้าถึงบริการบนพอร์ต `8080` ของ Guest ผ่านพอร์ต `8080` ของ Host ได้ หากบริการและ Network ถูกตั้งค่าอย่างถูกต้อง

สามารถกำหนด Protocol เป็น UDP ได้:

```ruby
config.vm.network "forwarded_port",
  guest: 5353,
  host: 5353,
  protocol: "udp"
```

หาก Host Port ถูกใช้งานอยู่ สามารถใช้ `auto_correct: true` เพื่อให้ Vagrant พยายามเลือก Host Port อื่นที่ว่าง:

```ruby
config.vm.network "forwarded_port",
  guest: 8080,
  host: 8080,
  auto_correct: true
```

#### Public Network

Public Network ใช้เชื่อม VM เข้ากับ Network ภายนอก เช่น LAN ผ่านกลไกของ Provider และการตั้งค่าของระบบ โดยอาจต้องเลือก Network Interface และตั้งค่า IP Address เพิ่มเติม

### 6. Multiple Virtual Machines

Vagrant สามารถกำหนด VM หลายเครื่องภายใน `Vagrantfile` เดียวได้:

```ruby
Vagrant.configure("2") do |config|
  config.vm.define "web" do |web|
    web.vm.box = "bento/ubuntu-24.04"
    web.vm.hostname = "web"
  end

  config.vm.define "redis" do |redis|
    redis.vm.box = "bento/ubuntu-24.04"
    redis.vm.hostname = "redis"
  end
end
```

- `web` และ `redis` เป็นชื่อ VM ที่กำหนดภายใน Vagrant
- `web` และ `redis` ใน Block เป็นตัวแปรที่ใช้ตั้งค่า VM แต่ละเครื่อง
- แต่ละ VM สามารถมี Network, Provisioning และ Resource Configuration ของตัวเองได้

การสร้าง VM หลายเครื่องไม่ได้ทำให้ VM เหล่านั้นสื่อสารกันได้โดยอัตโนมัติ ต้องกำหนด Network และการเชื่อมต่อให้เหมาะสมด้วย

### 7. Provider Configuration

Provider Configuration ใช้กำหนดคุณสมบัติของ VM ที่ขึ้นอยู่กับ Hypervisor หรือ Provider ที่เลือกใช้

ตัวอย่างสำหรับ VirtualBox:

```ruby
config.vm.provider "virtualbox" do |vb|
  vb.memory = 2048
  vb.cpus = 2
end
```

ตัวอย่างนี้กำหนด Memory เป็น 2048 MB และ CPU เป็น 2 Cores โดยต้องตรวจสอบว่าค่าและตัวเลือกที่ใช้รองรับโดย Provider และเวอร์ชันที่ติดตั้ง

### 8. Environment Variables and Secrets

สามารถอ่าน Environment Variables จาก Environment ที่ใช้รัน Vagrant ผ่าน Ruby ได้:

```ruby
secret_token = ENV["KUBE_TOKEN"]
```

หากใช้ Plugin เช่น `vagrant-env` ต้องติดตั้งและตั้งค่าตามเอกสารของ Plugin ก่อนใช้งาน

ข้อควรระวัง:

- ไม่ควร Hardcode Secret ลงใน `Vagrantfile`
- ไม่ควร Commit ไฟล์ที่มี Secret ลง Git
- ควรตรวจสอบว่าตัวแปรที่จำเป็นมีค่าก่อนใช้งาน
- การอ่าน Environment Variable บน Host ไม่ได้หมายความว่า Secret จะถูกส่งเข้า Guest โดยอัตโนมัติ

## Common Workflow

### 1. Initialize

```bash
vagrant init bento/ubuntu-24.04
```

สร้าง `Vagrantfile` เริ่มต้น

### 2. Start the Environment

```bash
vagrant up
```

สร้างหรือเริ่ม VM ตาม Configuration

### 3. Access the VM

```bash
vagrant ssh
```

เชื่อมต่อเข้า VM ผ่าน SSH หากมี VM หลายเครื่อง สามารถระบุชื่อ VM ได้ เช่น:

```bash
vagrant ssh web
```

### 4. Check VM Status

```bash
vagrant status
```

ตรวจสอบสถานะของ VM ที่ Vagrant จัดการอยู่

### 5. Run Provisioning

```bash
vagrant provision
```

รัน Provisioner ที่กำหนดไว้กับ VM

### 6. Stop the Environment

ปิด VM:

```bash
vagrant halt
```

พักสถานะ VM:

```bash
vagrant suspend
```

กลับมาทำงานต่อจากสถานะพัก:

```bash
vagrant resume
```

### 7. Destroy the Environment

```bash
vagrant destroy
```

ลบ VM ที่ Vagrant จัดการอยู่ โดยข้อมูลที่เก็บอยู่ภายใน VM อาจสูญหาย ควรสำรองข้อมูลที่จำเป็นก่อนดำเนินการ

### Important Notes

- `vagrant up` ไม่จำเป็นต้องสร้าง VM ใหม่ทุกครั้ง หาก VM มีอยู่แล้ว Vagrant อาจเริ่มเครื่องเดิม
- `vagrant provision` ใช้รัน Provisioning อีกครั้ง แต่ผลลัพธ์ขึ้นอยู่กับ Script และ Provisioner ที่กำหนด
- การทำ Environment ให้ Reproducible ต้องควบคุมทั้ง Vagrantfile, Box Version, Provisioning Script และ Dependencies ที่เกี่ยวข้อง
- ควรตรวจสอบเอกสารทางการของ Vagrant และ Provider เมื่อใช้ Configuration ที่ขึ้นอยู่กับเวอร์ชัน
