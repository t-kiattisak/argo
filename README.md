# Argo CD GitOps Production Configuration Repository

คลังเก็บการกำหนดค่าระบบ (GitOps Repository) สำหรับติดตั้ง ปรับแต่ง และจัดการแอปพลิเคชันด้วย Argo CD ในระดับ Production

---

## 📂 โครงสร้างโฟลเดอร์ของคลังเก็บข้อมูล

```bash
├── README.md                  # คู่มือแนะนำการใช้งานระบบ
├── argocd-install/            # Manifests สำหรับการติดตั้งและคอนฟิก Argo CD (HA Mode)
│   ├── kustomization.yaml     # Kustomize bundling สำหรับการจัดการ Helm Chart
│   ├── namespace.yaml         # สร้าง namespace 'argocd'
│   └── values-ha.yaml         # Helm Values สำหรับติดตั้ง Argo CD แบบ High Availability (HA)
├── infrastructure/            # ส่วนประกอบพื้นฐานของ Cluster (Cluster-wide Services)
│   ├── kustomization.yaml
│   ├── cert-manager/          # จัดการ SSL/TLS Certificates แบบอัตโนมัติ
│   ├── ingress-nginx/         # Ingress Controller สำหรับกำหนดทราฟฟิกเข้าสู่ Cluster
│   ├── external-secrets/      # ดึง Kubernetes Secrets จาก External Cloud Providers (AWS/GCP)
│   └── sealed-secrets/        # ทางเลือก: เข้ารหัสลับใน Git (กรณีไม่ใช้ Cloud Provider)
└── apps/                      # คอนฟิกแอปพลิเคชันทั้งหมดที่ Deploy ใน Cluster
    ├── base/                  # เทมเพลตมาตรฐานของแอป (Deployment, Service, ingress)
    │   └── my-api/
    ├── overlays/              # ทับซ้อนตัวแปรคอนฟิกเฉพาะตามสภาพแวดล้อม (Staging, Production)
    │   └── my-api-prod/       # ปรับแต่งคอนฟิกเฉพาะของ Production Environment (เช่น replicas, resource limits)
    └── templates/             # ApplicationSet สำหรับให้ Argo CD ค้นหาแอปมาติดตั้งอัตโนมัติ
        └── appset-git-generator.yaml
```

---

## 🚀 ขั้นตอนการติดตั้งและการนำไปใช้งาน

### 1. การติดตั้ง Argo CD ในโหมด High Availability (HA)

เราจะใช้ Kustomize เพื่อจัดการสร้าง Namespace และนำ Helm Chart ของ Argo CD มาติดตั้งพร้อมระบุค่า Config สำหรับระดับ Production (`values-ha.yaml`)

```bash
# ตรวจสอบการสร้าง Resources ก่อนนำไปรันจริง
kubectl kustomize argocd-install/ --enable-helm

# สั่งติดตั้งไปยัง Kubernetes Cluster ของคุณ
kubectl apply -k argocd-install/ --enable-helm
```

*หมายเหตุ: ใน `argocd-install/values-ha.yaml` ได้ทำการปิดการใช้งาน admin user (`admin.enabled: "false"`) ไว้เป็นค่าแนะนำ เพื่อความปลอดภัยใน Production ให้เชื่อมต่อ SSO ก่อนใช้งานจริง แต่ถ้าต้องการทดสอบในช่วงแรก สามารถเปลี่ยนค่าในไฟล์เป็น `"true"` หรือเข้าหน้าเว็บด้วย Password ของ admin ซึ่งสามารถดึงได้โดยคำสั่ง:*
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

---

### 2. การติดตั้งระบบโครงสร้างพื้นฐาน (Infrastructure Components)

คุณสามารถติดตั้ง Ingress Controller, Cert Manager, และ Secret Management เพื่อซัพพอร์ตระบบด้วยคำสั่งเดียว:

```bash
# ทำการรัน Kustomize เพื่อติดตั้ง infra ทั้งหมด
kubectl apply -k infrastructure/
```

> 💡 **ข้อแนะนำ**: หากใช้ AWS, GCP หรือ Key Vault ให้ตรวจสอบการตั้งค่า Service Account IAM Roles ในโฟลเดอร์ `infrastructure/external-secrets/` เพื่อให้ระบบมีสิทธิ์เข้าไปดึงค่า Keys ได้อย่างปลอดภัย

---

### 3. การ Onboard แอปพลิเคชัน (Application Onboarding)

สำหรับการ Deploy แอปพลิเคชันเข้าไปใน Cluster เราขอแนะนำให้ใช้วิธี **ApplicationSet Pattern** ซึ่งจะทำหน้าที่สร้างทรัพยากร Application ของ Argo CD อัตโนมัติเมื่อพบโฟลเดอร์ใหม่เกิดขึ้นใน `apps/overlays/`

1. นำไฟล์ ApplicationSet ไปติดตั้งบน Cluster (Management Cluster):
   ```bash
   kubectl apply -f apps/templates/appset-git-generator.yaml
   ```
2. เมื่อต้องการเพิ่มแอปใหม่ ให้ทำโครงสร้างตามตัวอย่าง `apps/base/my-api` และ `apps/overlays/my-api-prod`
3. Argo CD จะตรวจหาโฟลเดอร์ใน `apps/overlays/*` แล้วทำการสร้างแอปและ Deploy ให้อัตโนมัติ (Auto-Sync)

---

## 🔒 แนวทางปฏิบัติเพื่อความปลอดภัยระดับ Production

1. **ห้าม Commit plain-text Secrets**: ให้ใช้ `External Secrets Operator` ดึงจาก Cloud Key Provider หรือใช้ `Sealed Secrets` ทำการเข้ารหัสก่อน Commit
2. **SSO Integration**: ตั้งค่า OIDC ในไฟล์ `argocd-cm` ภายใต้ `argocd-install/values-ha.yaml` (มีเทมเพลตตัวอย่างระบุไว้ด้านใน)
3. **RBAC Policy**: ควบคุมสิทธิ์ของทีมงาน โดยจำกัดให้ Developers มีสิทธิ์ระดับ Read/Sync ได้ใน namespace ของตัวเองเท่านั้น และผู้ดูแลระบบหลักมีสิทธิ์ระดับ Admin
4. **Network Policies**: ควรจำกัดทางเข้าออกของ Argo CD Server และ Redis Sentinel ไม่ให้เปิดรับ Public Connection โดยไม่มีความจำเป็น
5. **Git Connection**: เชื่อมต่อไปยัง Git Provider (เช่น GitHub) แนะนำให้ใช้ **GitHub App Authentication** แทนการแชร์ SSH Key เพื่อสิทธิ์ที่กระชับและตรวจสอบได้ง่าย
