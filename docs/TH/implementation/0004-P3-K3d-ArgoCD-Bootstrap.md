# P3 — K3d Cluster and Argo CD Bootstrap

## Context

P3 ใช้ K3d สร้าง Kubernetes cluster ใน Docker แล้วติดตั้ง Argo CD ใน namespace `argocd` จากนั้นสร้าง namespace `dev` สำหรับ workload ที่ Argo CD จะจัดการ Makefile รวมคำสั่งเหล่านี้ไว้เป็น targets เพื่อให้เรียก setup ได้จากขั้นตอนเดียว

Source: [`p3/Makefile`](../../../p3/Makefile), [`p3/confs/wil_app.yaml`](../../../p3/confs/wil_app.yaml), [`p3/scripts/argo.sh`](../../../p3/scripts/argo.sh)

## 1. Cluster Port Mapping

**Source: `p3/Makefile:1–15`**

```makefile
ARGO_PORT = 8080
WIL_PORT = 8888

all: create-cluster dev argo

create-cluster:
	@ echo "Create K3d Cluster"
	@ k3d cluster create demo --servers 1 --agents 1 \
	--port "$(ARGO_PORT):$(ARGO_PORT)@loadbalancer" \
	--port "$(WIL_PORT):$(WIL_PORT)@loadbalancer" \
	--wait > /dev/null
```

Target `all` ประกาศ dependency ของการสร้าง cluster, namespace `dev` และ Argo CD bootstrap ตามลำดับที่เขียนไว้ (หากเรียก Make โดยไม่ใช้ parallel mode) คำสั่ง K3d สร้าง cluster ชื่อ `demo` มี server หนึ่ง node และ agent หนึ่ง node และ map host ports `8080` กับ `8888` ไปยัง K3d load balancer

การ map port เป็นคนละชั้นกับการประกาศ Kubernetes Service: K3d เปิดช่องจาก host ไปยัง load balancer container ส่วน Service ภายใน cluster กำหนดว่าจะส่ง traffic ต่อไปที่ Pod อย่างไร

## 2. Install and Expose Argo CD

**Source: `p3/Makefile:17–36`**

```makefile
@ kubectl create namespace argocd > /dev/null 2>&1
@ kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml > /dev/null
@ kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=300s > /dev/null
@ kubectl patch svc argocd-server -n argocd --type merge -p '{"spec": {"type": "LoadBalancer", "ports": [{"name": "https", "port": 8080, "targetPort": 8080, "protocol": "TCP"}]}}'
@ kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=90s
@ until curl -sk https://localhost:8080 > /dev/null 2>&1; do sleep 1; done
```

Makefile ติดตั้ง Argo CD จาก remote `stable` manifest, รอ Deployment พร้อม แล้ว patch Service `argocd-server` ให้เป็น `LoadBalancer` ที่พอร์ต 8080 เพื่อให้สอดคล้องกับ host-port mapping ที่สร้างไว้ก่อนหน้า จากนั้นรอให้ HTTPS endpoint ที่ `localhost:8080` ตอบสนอง

เพราะ URL ใช้ `stable` แทน version/tag ที่ pin ไว้ ผลลัพธ์ของการติดตั้งอาจเปลี่ยนเมื่อ upstream stable เปลี่ยน นอกจากนี้การ redirect output และปิด error ของ `kubectl create namespace` ทำให้ log สะอาดขึ้น แต่ก็ลดรายละเอียดเมื่อคำสั่งมีปัญหา

## 3. Log In with the Initial Admin Secret

**Source: `p3/Makefile:34–36`**

```makefile
@ PASSWORD=$$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d); \
argocd login localhost:8080 --insecure --username admin --password "$$PASSWORD"
```

คำสั่งอ่าน password จาก Kubernetes Secret, ถอด Base64 และส่งให้ Argo CD CLI เพื่อ login `--insecure` บอก CLI ให้ยอมรับ TLS certificate โดยไม่ตรวจสอบความถูกต้องตามปกติ เหมาะกับการทดสอบ local ตามที่โค้ดตั้งไว้ แต่ไม่ควรตีความว่าเป็นรูปแบบ TLS ที่เหมาะกับ production

## 4. Application Creation Has a Line-Continuation Defect

**Source: `p3/Makefile:38–47`**

```makefile
@ argocd app create wil-app \
    --repo https://github.com/lockdoor/42_inception_of_thing.git \
    --revision main
    --path p3/confs \
    --dest-server https://kubernetes.default.svc \
    --dest-namespace dev
```

บรรทัด `--revision main` ไม่มี `\` ต่อท้าย ดังนั้น shell จบคำสั่ง `argocd app create` ก่อนถึง `--path`, `--dest-server` และ `--dest-namespace` บรรทัดเหล่านี้จะถูกเรียกเป็นคำสั่งใหม่ แทนที่จะเป็น arguments ของ `argocd app create` ทำให้ target `argo` มีข้อผิดพลาดที่เห็นได้จาก source และควรแก้/ทดสอบก่อนอ้างว่า bootstrap นี้สำเร็จ

ใน `p3/scripts/argo.sh:7–11` มีการสร้าง Application อีกชุดหนึ่งด้วยค่า repo/path ต่างจาก Makefile และระบุ `--path ./confs/wil_app.yaml` ซึ่งชี้ไปที่ไฟล์ manifest เดี่ยว ขณะที่ `argocd app create --path` ใช้ path ของ app directory; หากต้องการระบุไฟล์ manifest ให้ตรวจวิธี `--file` ของ CLI แทน จึงควรระบุให้ชัดว่า workflow ที่ทดสอบใช้ entry point ใด อย่าสมมติว่าทั้งสองวิธีเทียบเท่ากันจนกว่าจะตรวจผล

ใน `p3/confs/wil_app.yaml:5,30` namespace ถูก comment ไว้ ขณะที่ Makefile ตั้ง `--dest-namespace dev` ประวัติ Git มี commit ชื่อ `fixed: p3 will app namespace` และ diff เปลี่ยน resources ให้ไม่ hard-code namespace ([commit 7a8c9b9](https://github.com/lockdoor/42_inception_of_thing/commit/7a8c9b98b79abca29d00ea47ba5d4a96bb476b96)). นี่เป็นหลักฐานจากประวัติที่สอดคล้องกับการให้ Application destination กำหนด namespace แต่ยังไม่ใช่ผล runtime test

## 5. Git Change Detection Is Not the Same as Auto-Sync

README ระบุว่า Argo CD จะ deploy หลัง push การเปลี่ยนแปลงไป GitHub แต่คำสั่ง `argocd app create` ใน Makefile ไม่ได้กำหนด `--sync-policy automated` และ `wil_app.yaml` เป็น Deployment/Service manifest ไม่ใช่ Argo CD Application manifest ที่มี `spec.syncPolicy.automated`

ดังนั้น source ที่เห็นยังไม่พิสูจน์ว่า automatic sync ถูกเปิดใช้งาน แม้ Argo CD จะตรวจพบ revision ใหม่ได้ก็ตาม เอกสาร Argo CD ระบุว่าต้องกำหนด automated sync policy อย่างชัดเจน [Automated Sync Policy](https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/).

นอกจากนี้ `kubectl wait --for=condition=synced app/wil-app -n dev` มีจุดน่าสงสัยสองอย่าง: Application resource โดยปกติอยู่ใน namespace ของ Argo CD (มักเป็น `argocd`) เว้นแต่ตั้งค่า application namespace เพิ่มเติม ขณะที่ `--dest-namespace dev` ใช้กำหนด namespace ปลายทางของ workload ไม่ใช่ namespace ที่เก็บ Application; อีกทั้ง sync status อยู่ใน `status.sync.status` จึงไม่ควรสมมติว่าเป็น Kubernetes condition แบบเดียวกับ Deployment ต้องตรวจ namespace และวิธีรอ sync จริงก่อนใช้คำสั่งนี้เป็นหลักฐาน

## Edge Cases and Limitations

- Makefile สร้าง Application command ไม่ถูกต่อบรรทัดครบ จึงไม่ควรถือว่า `make all` ผ่านได้จากการอ่าน source เพียงอย่างเดียว
- `stable` manifest ไม่ pin Argo CD version
- namespace creation suppresses errors; ควรตรวจ namespace และทรัพยากรจริงแยก
- README อธิบาย intended workflow แต่ configuration ที่เห็นยังไม่เปิด auto-sync อย่างชัดเจน
- `p3/confs/wil_app.yaml` ต้องถูกอ่านจาก Git revision/path ที่ Application ใช้จริง มิฉะนั้นการเปลี่ยน image ใน local working tree ไม่ได้หมายความว่า Argo CD จะเห็นการเปลี่ยนแปลงนั้น

## Verification

```bash
k3d cluster list
kubectl get ns argocd dev
kubectl get deploy,svc -n argocd
argocd app list
argocd app get wil-app
kubectl get applications.argoproj.io -A -o yaml
```

ตรวจ source repo/path/revision, destination namespace, sync policy และ sync status ของ Application จริงก่อนยืนยัน workflow ห้ามใช้ข้อความใน README เพียงอย่างเดียวเป็นหลักฐานว่า automatic deployment ทำงานแล้ว

## References

- [Argo CD `argocd app create`](https://argo-cd.readthedocs.io/en/latest/user-guide/commands/argocd_app_create/)
- [Argo CD Automated Sync Policy](https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/)
- [Git history: P3 application namespace change](https://github.com/lockdoor/42_inception_of_thing/commit/7a8c9b98b79abca29d00ea47ba5d4a96bb476b96)
- Source: [`Makefile`](../../../p3/Makefile), [`wil_app.yaml`](../../../p3/confs/wil_app.yaml), [`argo.sh`](../../../p3/scripts/argo.sh), [`README.md`](../../../p3/README.md)
