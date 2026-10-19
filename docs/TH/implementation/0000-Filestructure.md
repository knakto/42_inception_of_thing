# Repository Structure

## Purpose

เอกสารนี้เป็นแผนที่สำหรับตามหา implementation ใน repository ไม่ได้อธิบายรายละเอียดทุกไฟล์ และไม่ใช่ลำดับการทำงานแทนเอกสาร Implementation Notes รายเรื่อง

เอกสารชุดนี้ครอบคลุม P1–P3 เท่านั้น ส่วนคำอธิบายกลไกและข้อจำกัดจะอยู่ในไฟล์เฉพาะเรื่อง เพื่อให้ผู้ตรวจตามจากคำอธิบายไปยังโค้ดที่ทำให้เกิดพฤติกรรมนั้นได้โดยตรง

## Repository Areas

| ส่วน | หน้าที่จากโค้ด | จุดเริ่มต้นในการอ่าน |
|---|---|---|
| `p1/` | สร้าง VM สำหรับ K3s Server และ Worker พร้อมติดตั้ง K3s และส่ง node token | [`p1/Vagrantfile`](../../../p1/Vagrantfile), [`server.sh`](../../../p1/scripts/server.sh), [`worker.sh`](../../../p1/scripts/worker.sh) |
| `p2/` | สร้าง K3s Server แล้ว deploy แอปตัวอย่างสามตัวด้วย ConfigMap, Deployment, Service และ Ingress | [`p2/scripts/server.sh`](../../../p2/scripts/server.sh), [`p2/confs/`](../../../p2/confs/) |
| `p3/` | สร้าง K3d cluster, ติดตั้ง Argo CD และประกาศแอปตัวอย่างที่อ่าน manifest จาก Git | [`p3/Makefile`](../../../p3/Makefile), [`wil_app.yaml`](../../../p3/confs/wil_app.yaml), [`argo.sh`](../../../p3/scripts/argo.sh) |
| `docs/TH/` และ `docs/EN/` | เก็บเอกสารภาษาไทยและภาษาอังกฤษ แยก Concepts, ADRs, Requirements และ Implementation Notes | โฟลเดอร์ `docs/TH/` และ `docs/EN/` |

## How the Implementation Notes Are Split

- [0001 — P1: K3s Provisioning](0001-P1-K3s-Provisioning.md) — การสร้าง VM, การกำหนด network identity, การส่ง token และการเตรียม kubeconfig
- [0002 — P2: ConfigMap and Sidecar](0002-P2-ConfigMap-Sidecar.md) — การเปลี่ยนค่าจาก ConfigMap ให้เป็นหน้าเว็บที่ Nginx ให้บริการผ่าน shared volume
- [0003 — P2: Service and Ingress Routing](0003-P2-Service-Ingress-Routing.md) — การเลือก Pod ด้วย labels และการ route request ด้วย Host/path
- [0004 — P3: K3d and Argo CD Bootstrap](0004-P3-K3d-ArgoCD-Bootstrap.md) — การสร้าง cluster, ติดตั้ง Argo CD และการสร้าง Application
- [0005 — P3: Application Exposure](0005-P3-Application-Exposure.md) — การเชื่อม host ports, K3d load balancer, Kubernetes Service และ container port

## Reading Principles

Implementation Notes อ้างอิง source file และยกเฉพาะ code ที่จำเป็นต่อการอธิบายกลไก ไม่คัดลอกไฟล์ทั้งไฟล์ และไม่ถือว่าการมี configuration เป็นหลักฐานว่า runtime ทำงานสำเร็จ

สถานะในร่างชุดนี้มาจากการอ่าน source code แบบ static review เท่านั้น คำสั่งในหัวข้อ Verification เป็นวิธีที่เสนอให้ใช้ตรวจ ไม่ใช่ผลทดสอบที่รันยืนยันแล้ว หากข้อความใน README ระบุเป้าหมายที่ยังไม่เห็น configuration รองรับ จะระบุแยกจากพฤติกรรมที่ยืนยันได้จากโค้ด

## References

- [Project requirements](../42_project_requirement.md)
- [ADR-0001 — Host environment](../adrs/ADR-0001-host-environment.md)
