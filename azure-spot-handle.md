สำหรับการติดตั้ง **Azure Node Termination Handler (NTH)** ผ่าน Helm นั้นค่อนข้างตรงไปตรงมาครับ แต่จุดที่สำคัญที่สุดคือการตั้งค่าให้มันไปรันอยู่บน **Spot Nodes** ของคุณได้ (เพราะปกติ Spot Node จะมี Taint กันไว้)

นี่คือขั้นตอนและคำสั่งแบบ Step-by-Step ครับ

---

## 1. เพิ่ม Helm Repository
ขั้นแรกต้องเพิ่ม Repo ของ Microsoft เพื่อดึง Chart ล่าสุดมาใช้ครับ

```bash
helm repo add microsoft https://microsoft.github.io/azure-node-termination-handler/
helm repo update
```

---

## 2. เตรียมไฟล์การตั้งค่า (`values.yaml`)
ผมแนะนำให้สร้างไฟล์ชื่อ `nth-values.yaml` เพื่อกำหนดค่าให้ NTH สามารถเข้าไปทำงานบน Spot Instance ได้อย่างถูกต้องครับ

```yaml
# nth-values.yaml

# อนุญาตให้ NTH รันบน Node ที่ติด Taint (เช่น Spot Nodes)
tolerations:
  - key: "kubernetes.azure.com/scalesetpriority"
    operator: "Equal"
    value: "spot"
    effect: "NoSchedule"

# เลือกให้ NTH รันเฉพาะบน Spot Nodes เท่านั้น (ประหยัด Resource บน On-demand)
nodeSelector:
  kubernetes.azure.com/scalesetpriority: spot

# การตั้งค่าการแจ้งเตือน
config:
  # ระยะเวลาที่รอให้ Pod อื่นๆ ย้ายออก (Grace Period)
  # ปกติ Azure ให้ 30 วินาที เราอาจตั้งไว้ที่ 25 เพื่อความปลอดภัย
  taintNodeBeforeDrain: true
  excludeFromLoadBalancer: true
```

---

## 3. คำสั่งติดตั้ง (Installation)
ใช้คำสั่งนี้เพื่อติดตั้งลงใน Namespace ที่ต้องการ (แนะนำ `kube-system` เพื่อความเป็นระเบียบ)

```bash
# สร้าง Namespace (ถ้ายังไม่มี)
kubectl create namespace kube-system --dry-run=client -o yaml | kubectl apply -f -

# ติดตั้งด้วย Helm
helm install azure-nth microsoft/azure-node-termination-handler \
    --namespace kube-system \
    -f nth-values.yaml
```

---

## 4. วิธีการตรวจสอบ (Verification)
หลังจากติดตั้งเสร็จ คุณควรตรวจสอบว่า Pod ของ NTH รันอยู่บนทุก Node ที่เป็น Spot หรือไม่:

1.  **เช็กสถานะ Pod:**
    ```bash
    kubectl get pods -n kube-system -l app.kubernetes.io/name=azure-node-termination-handler
    ```
2.  **ดู Log เพื่อยืนยันว่ามันกำลังฟังสัญญาณจาก Azure อยู่:**
    ```bash
    # เลือกมาสัก 1 Pod เพื่อดู log
    kubectl logs -f <pod-name-จากคำสั่งข้างบน> -n kube-system
    ```
    *ใน Log คุณควรจะเห็นข้อความประมาณว่า `Scheduled Events API is reachable`*

---

## สรุปการทำงานของ NTH ใน Cluster ของคุณ

| ส่วนประกอบ | หน้าที่ในระบบนี้ |
| :--- | :--- |
| **DaemonSet** | NTH จะรัน 1 ตัวต่อ 1 Spot Node เพื่อคอยเฝ้าบ้าน |
| **IMDS Endpoint** | NTH จะยิงไปถาม IP `169.254.169.254` (Azure Internal) ทุกๆ วินาที |
| **Cordon & Drain** | เมื่อได้รับสัญญาณ `Preempt`, NTH จะสั่งให้ K8s ย้ายคนอื่นออกทันที |

---

### **ข้อแนะนำเพิ่มเติม (Pro-tip)**
หากคุณใช้ **AKS Managed Node Pools** รุ่นใหม่ๆ (GA ตั้งแต่ช่วงปี 2024 เป็นต้นมา) บางครั้ง Azure จะเปิดฟีเจอร์ **Spot Termination Handler** มาให้แบบ Built-in เลย (ไม่ต้องลง Helm เพิ่ม) คุณสามารถเช็กได้โดยการดูที่ Node Pool property:

```bash
az aks nodepool show --cluster-name <cluster> --resource-group <rg> --name <pool-name> --query "spotConfig"
```

ถ้าค่า `enableSpotMaxPrice` และการตั้งค่าพื้นฐานดูเรียบร้อยแล้ว NTH แบบ Manual (Helm) นี้จะเป็นตัวช่วยเสริมความมั่นใจได้ดีมากครับ

คุณต้องการให้ผมช่วยดูวิธีการเขียน **PreStop Lifecycle Hook** ในแอปของคุณเพื่อให้มัน Shutdown ได้นุ่มนวลขึ้นระหว่างที่ NTH กำลังทำงานไหมครับ?