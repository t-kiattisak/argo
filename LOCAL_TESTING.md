# คู่มือการทดสอบ Argo CD แบบ Local (ด้วย Docker/Kubernetes)

การทดสอบ GitOps Repository นี้บนเครื่องคอมพิวเตอร์ของคุณ (Local Machine) สามารถทำได้โดยง่ายผ่าน **Docker Desktop** หรือ **Kind (Kubernetes in Docker)** โดยเราได้เตรียมไฟล์คอนฟิกน้ำหนักเบาสำหรับทดสอบไว้ให้แล้ว (`values-local.yaml`)

---

## 🛠️ สิ่งที่ต้องมีก่อนเริ่ม (Prerequisites)

1. **Docker Desktop** ติดตั้งและเปิดใช้งานอยู่
2. **Kubernetes Cluster (Local)**:
   * **ทางเลือกที่ 1 (แนะนำสำหรับ Mac/Windows)**: เปิดใช้งาน Kubernetes ใน Docker Desktop (ไปที่ Settings -> Kubernetes -> ติ๊กถูก "Enable Kubernetes" -> Apply & Restart)
   * **ทางเลือกที่ 2 (สำหรับสาย Terminal)**: ติดตั้ง [Kind](https://kind.sigs.k8s.io/) แล้วสร้างคลัสเตอร์ด้วยคำสั่ง:
     ```bash
     kind create cluster --name argo-test
     ```
3. **kubectl**: CLI สำหรับจัดการ Kubernetes
4. **Helm**: CLI สำหรับดาวน์โหลดและจัดการ Helm Chart

---

## 🚀 ขั้นตอนการทดสอบ

### 1. ติดตั้ง Argo CD แบบ Local (ลดขนาดสเปกเพื่อประหยัด RAM)

เปิด Terminal ในโฟลเดอร์โปรเจกต์นี้ แล้วรันคำสั่ง:

```bash
# 1. สร้าง Namespace สำหรับ Argo CD
kubectl create namespace argocd

# 2. เพิ่ม Helm Repository ของ Argo
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# 3. ติดตั้ง Argo CD โดยใช้ config สำหรับทดสอบในเครื่อง (ปิด HA และเปิดใช้งาน Admin User)
helm upgrade --install argocd argo/argo-cd \
  --namespace argocd \
  -f argocd-install/values-local.yaml
```

ตรวจสอบสถานะของ Pods จนกระทั่งเปลี่ยนเป็น `Running` ทั้งหมด:
```bash
kubectl get pods -n argocd
```

---

## 🔓 การลบทรัพยากรออกหลังทดสอบเสร็จ

เมื่อทดสอบเสร็จสิ้นและต้องการคืนทรัพยากร RAM/CPU ให้กับคอมพิวเตอร์ของคุณ:

```bash
# ลบแอปพลิเคชันทดสอบ
kubectl delete namespace my-api-prod

# ถอนการติดตั้ง Argo CD
helm uninstall argocd -n argocd
kubectl delete namespace argocd

# หากใช้ Kind คลัสเตอร์ สามารถลบคลัสเตอร์ทิ้งได้ทันที
kind delete cluster --name argo-test
```
