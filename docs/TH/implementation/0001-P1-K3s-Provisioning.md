# P1 — K3s Provisioning Across Two VMs

## Context

P1 แยก K3s Server และ Worker เป็น VM คนละเครื่อง โดยใช้ Vagrant จัดการ lifecycle ของ VM และใช้ shell provisioner ติดตั้ง K3s ภายในแต่ละเครื่อง กลไกสำคัญคือ Worker ต้องรู้ทั้งปลายทางของ API Server และ token ที่ใช้เข้าร่วม cluster ขณะที่ทั้งสองเครื่องต้องใช้ network identity ที่ไม่ซ้ำกัน

แหล่งโค้ด: [`p1/Vagrantfile`](../../../p1/Vagrantfile), [`p1/scripts/server.sh`](../../../p1/scripts/server.sh), [`p1/scripts/worker.sh`](../../../p1/scripts/worker.sh)

## 1. VM Network and Shared Directory

**Source: `p1/Vagrantfile:4–24`**

```ruby
config.vm.define "pnamnilS" do |pnamnilS|
    pnamnilS.vm.hostname = "pnamnilS"
    pnamnilS.vm.network "private_network", ip: "192.168.56.110"
    pnamnilS.vm.provider "virtualbox" do |vb|
        vb.cpus = "2"
        vb.memory = "2048"
    end
    pnamnilS.vm.synced_folder "./confs", "/home/vagrant/confs", create:true
    pnamnilS.vm.provision "shell", path: "./scripts/server.sh"
end

config.vm.define "pnamnilSW" do |pnamnilSW|
    pnamnilSW.vm.hostname = "pnamnilSW"
    pnamnilSW.vm.network "private_network", ip: "192.168.56.111"
    pnamnilSW.vm.provider "virtualbox" do |vb|
        vb.cpus = "1"
        vb.memory = "1024"
    end
    pnamnilSW.vm.synced_folder "./confs", "/home/vagrant/confs", create:true
    pnamnilSW.vm.provision "shell", path: "./scripts/worker.sh"
end
```

`private_network` กำหนด IP ให้ VM แต่ละเครื่อง ส่วน `synced_folder` สองรายการชี้ไปยัง host directory เดียวกัน (`./confs`) แล้ว mount เข้า VM ที่ path เดียวกัน (`/home/vagrant/confs`) จึงเป็นช่องทางที่ `server.sh` เขียน token แล้ว `worker.sh` อ่านได้โดยไม่ต้องส่งไฟล์ผ่าน network protocol เพิ่มอีกตัว

การใช้ชื่อ interface `eth1` ในสคริปต์ทั้งคู่ไม่ได้ทำให้เป็น interface เดียวกันทางกายภาพ แต่หมายถึง interface ที่มีชื่อนั้นภายใน VM แต่ละเครื่อง ประวัติ Git รุ่นก่อนมี commit ชื่อ `fixed: ip in worker node should bind interface eth1 for rigth communication` ซึ่งสนับสนุนว่าการระบุ interface เกี่ยวข้องกับการสื่อสารระหว่าง node แต่ชื่อ commit เพียงอย่างเดียวไม่ใช่หลักฐานว่าทุกกรณีของ network ได้รับการทดสอบแล้ว ([commit af73b33](https://github.com/lockdoor/42_inception_of_thing/commit/af73b3337b923c8bcd27bfcf0133ed84095badfd))

## 2. Server Identity and Flannel Interface

**Source: `p1/scripts/server.sh:3–5`**

```bash
curl -sfL https://get.k3s.io | \
    INSTALL_K3S_EXEC="server --node-ip=192.168.56.110 --flannel-iface=eth1" sh -s -
```

`--node-ip` เลือก IP ที่ K3s ใช้เป็น node address ของเครื่องนี้ ส่วน `--flannel-iface` ระบุ interface ที่ Flannel ใช้สำหรับ node networking การกำหนดทั้งสองค่าแยกกันช่วยไม่ให้การเลือก node address ต้องพึ่งการเดา interface ที่เหมาะสมจาก routing table ของ VM

Worker กำหนดค่าเป็น `.111` และ `eth1` ของตัวเองใน `worker.sh:9–12` ค่าของ Worker จึงไม่ควรคัดลอก IP ของ Server มาใช้ซ้ำ

## 3. Token Handoff and Worker Join

**Source: `p1/scripts/server.sh:13–14`**

```bash
cat /var/lib/rancher/k3s/server/node-token >> /home/vagrant/confs/token
```

**Source: `p1/scripts/worker.sh:3–15`**

```bash
if [ ! -f /home/vagrant/confs/token ]; then
    echo "No token found, please check pnamnilS first"
    exit 1
fi

curl -sfL https://get.k3s.io | K3S_URL=https://192.168.56.110:6443 \
    K3S_TOKEN=$(cat /home/vagrant/confs/token) \
    INSTALL_K3S_EXEC="agent --node-ip=192.168.56.111 --flannel-iface=eth1" sh -s -

rm /home/vagrant/confs/token
```

มีข้อมูลคนละชนิดที่ Worker ต้องใช้:

- `K3S_URL` คือปลายทาง K3s API Server ที่ Worker จะติดต่อเพื่อ join cluster (`192.168.56.110:6443`)
- `K3S_TOKEN` เป็นข้อมูลยืนยันตัวตนที่อ่านจาก shared folder
- `--node-ip` คือ IP identity ของ Worker เอง ไม่ใช่ IP ของ Server
- `--flannel-iface` เลือก interface สำหรับ Flannel บน Worker

การตรวจ `-f` ป้องกันไม่ให้สคริปต์เริ่มติดตั้ง Agent เมื่อยังไม่มีไฟล์ token แต่ไม่ได้ยืนยันว่า token ถูกต้อง, Server พร้อมรับ connection หรือ Worker ติดต่อ port 6443 ได้แล้ว

หลังติดตั้ง Worker ลบ `/home/vagrant/confs/token` ซึ่งเป็น path ที่แชร์กับ host directory ด้วย นี่เป็นการลบไฟล์ใน shared directory ไม่ใช่การลบ token จาก K3s Server

## 4. Kubeconfig for the Vagrant User

**Source: `p1/scripts/server.sh:7–11`**

```bash
mkdir -p /home/vagrant/.kube
sudo cp /etc/rancher/k3s/k3s.yaml /home/vagrant/.kube/config
sudo chown -R vagrant:vagrant /home/vagrant/.kube
echo 'export KUBECONFIG=/home/vagrant/.kube/config' >> /home/vagrant/.bashrc
```

Provisioning script ทำงานด้วยสิทธิ์สูง จึงคัดลอก kubeconfig จากตำแหน่งที่ K3s สร้างไว้ไปยัง home ของผู้ใช้ `vagrant` และเปลี่ยน owner เพื่อให้ผู้ใช้เข้าถึงไฟล์ได้ จากนั้นตั้ง `KUBECONFIG` ใน `.bashrc` ให้ shell ที่อ่านไฟล์นี้ใช้ config ตำแหน่งดังกล่าว

`KUBECONFIG` บอก `kubectl` ว่าจะอ่าน configuration จากที่ใด ไม่ได้สร้างหรือแก้ไข Kubernetes resources เอง นอกจากนี้สคริปต์ไม่ได้แก้ `server:` ภายใน kubeconfig ที่คัดลอกมา ดังนั้นก่อนใช้ config จากนอก VM ต้องตรวจ endpoint ในไฟล์จริง เพราะ kubeconfig เริ่มต้นของ K3s มักชี้ไปยัง loopback address

## Edge Cases and Limitations

- การ append token ด้วย `>>` อาจต่อ token ซ้ำ หาก provision `server.sh` หลายครั้งโดยไฟล์เดิมยังอยู่
- การตรวจเพียงว่า token file มีอยู่ไม่ได้ตรวจรูปแบบหรือความถูกต้องของ token
- การลบไฟล์จาก Worker กระทบ host shared folder ด้วย หากต้องการเก็บ token ไว้ตรวจสอบหรือใช้ซ้ำ วิธีนี้จะไม่เก็บไฟล์นั้นไว้
- script เขียนไฟล์ชื่อ `token` แต่ `.gitignore` ปัจจุบันระบุ `node-token` ไม่ใช่ `token`; อย่าสมมติว่า token file จะถูก ignore โดยอัตโนมัติ ตรวจ [`.gitignore`](../../../.gitignore) ก่อน commit และอย่า commit credential
- สคริปต์ไม่ได้ pin `INSTALL_K3S_VERSION` จึงเลือกเวอร์ชันจาก stable channel ณ เวลาที่ติดตั้ง ไม่ได้บังคับเวอร์ชันเดิมทุกครั้ง
- การ export `KUBECONFIG` ใน `.bashrc` ส่งผลต่อ shell ที่โหลดไฟล์ดังกล่าว ไม่ได้เปลี่ยนค่า environment ของทุก process ที่รันอยู่แล้ว

## Verification

ใช้คำสั่งต่อไปนี้ตรวจผลจริงหลัง VM provision สำเร็จ:

```bash
# บน Server VM
ip -br addr
sudo systemctl status k3s
kubectl get nodes -o wide

# บน Worker VM
ip -br addr
sudo systemctl status k3s-agent
```

ตรวจว่า node address ของ Server/Worker ตรงกับที่กำหนด และสถานะ node เป็น `Ready` ก่อนสรุปว่า join สำเร็จ อย่าแสดงค่า token เต็มใน log, screenshot หรือเอกสารที่แชร์สาธารณะ

## References

- [K3s networking options](https://docs.k3s.io/networking/basic-network-options)
- [K3s install script environment variables](https://docs.k3s.io/reference/env-variables)
- [Git history: interface binding change](https://github.com/lockdoor/42_inception_of_thing/commit/af73b3337b923c8bcd27bfcf0133ed84095badfd)
- Source: [`Vagrantfile`](../../../p1/Vagrantfile), [`server.sh`](../../../p1/scripts/server.sh), [`worker.sh`](../../../p1/scripts/worker.sh)
