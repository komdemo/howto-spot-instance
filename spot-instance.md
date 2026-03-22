สำหรับการใช้งาน **Azure Spot Node Pool** ใน AKS (Azure Kubernetes Service) นั้นถือเป็นท่ามาตรฐานที่ช่วยประหยัดเงินได้ถึง 60-90% เลยครับ แต่เนื่องจาก Spot Instance มีความเสี่ยงที่จะโดน "เรียกคืน" (Eviction) ได้ทุกเมื่อเมื่อ Azure ต้องการใช้ทรัพยากร 

ดังนั้น ระบบจะใส่ **Taint** มาให้เป็นค่าเริ่มต้นเพื่อป้องกันไม่ให้ Pod สำคัญๆ เผลอไปรันแล้วดับไปดื้อๆ ครับ

---

## 1. สิ่งที่ Azure เตรียมไว้ให้ (Auto-Generated)
เมื่อคุณสร้าง Spot Node Pool ใน AKS ระบบจะจัดการเรื่อง Label และ Taint ให้เราโดยอัตโนมัติ ดังนี้ครับ:

* **Label:** `kubernetes.azure.com/scalesetpriority=spot`
* **Taint:** `kubernetes.azure.com/scalesetpriority=spot:NoSchedule`

> **Note:** ตัว Taint นี้สำคัญมาก เพราะมันทำหน้าที่เป็น "ยาม" บอกว่า "ถ้า Pod ไหนไม่มีบัตรผ่าน (Toleration) ห้ามเข้ามาจอดที่ Node Pool นี้เด็ดขาด"

---

## 2. ขั้นตอนการสร้าง Spot Node Pool
หากคุณยังไม่มี สามารถใช้คำสั่ง Azure CLI เพื่อสร้างได้เลยครับ:

```bash
az aks nodepool add \
    --resource-group myResourceGroup \
    --cluster-name myAKSCluster \
    --name spotnodepool \
    --priority Spot \
    --eviction-policy Delete \
    --spot-max-price -1 \
    --node-vm-size Standard_DS2_v2 \
    --node-count 3
```

---

## 3. การเขียน Pod ให้รันบน Spot Instance
ในการเลือกโหนด เราจะใช้ 2 ส่วนทำงานร่วมกัน:
1.  **Tolerations:** เพื่อให้ Pod "ทนทาน" ต่อหนาม (Taint) ของ Spot Node ได้
2.  **Node Affinity (หรือ NodeSelector):** เพื่อ "ระบุเจาะจง" ว่าต้องลงที่เครื่อง Spot เท่านั้น (ไม่ไปแย่งที่เครื่อง On-Demand ราคาแพง)



### ตัวอย่าง YAML แบบละเอียด:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: batch-processor
spec:
  replicas: 5
  selector:
    matchLabels:
      app: batch-job
  template:
    metadata:
      labels:
        app: batch-job
    spec:
      # 1. บัตรผ่านสำหรับเข้า Node ที่ติด Taint 'spot'
      tolerations:
      - key: "kubernetes.azure.com/scalesetpriority"
        operator: "Equal"
        value: "spot"
        effect: "NoSchedule"

      containers:
      - name: processor
        image: my-batch-image:v1

      # 2. การระบุเจาะจงว่า "อยากได้" หรือ "ต้องเป็น" Spot Node
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.azure.com/scalesetpriority
                operator: In
                values:
                - spot
```

---

## 4. ข้อควรระวังและการจัดการที่ดี (Best Practices)

การใช้ Spot Instance ไม่ใช่แค่รันได้แล้วจบ แต่ต้องออกแบบแอปให้ "ตายแล้วเกิดใหม่" ได้เร็ว (Stateless) ด้วยครับ:

### การจัดการการโดนไล่ (Spot Termination Notice)
Azure จะส่งสัญญาณเตือนล่วงหน้า **30 วินาที** ก่อนจะดึง Node คืน คุณสามารถใช้เครื่องมือชื่อ **Azure Scheduled Events** หรือติดตั้งโปรแกรมเสริมใน Cluster เพื่อดักจับเหตุการณ์นี้แล้วทำ "Graceful Shutdown" ได้

### การผสมผสาน (Priority Mix)
อย่าเอาไข่ทั้งหมดไว้ในตะกร้าใบเดียวครับ:
* **On-Demand Pool:** ใช้รัน Database, Core API หรืออะไรที่ห้ามดับ
* **Spot Pool:** ใช้รัน Worker, Batch Processing, หรือ Dev/Test Environment

---

## สรุปคำสั่งตรวจสอบ
ถ้าอยากรู้ว่า Node ใน Cluster ของเรามี Label หรือ Taint อะไรอยู่บ้าง ให้ใช้คำสั่งนี้ครับ:

```bash
# ดู Taints
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints

# ดู Labels (กรองเฉพาะตัวที่เกี่ยวกับ spot)
kubectl get nodes --show-labels | grep scalesetpriority
```

คุณอยากให้ผมช่วยแนะนำเครื่องมือจำพวก **Node Termination Handler** เพื่อจัดการ Pod ให้ปลอดภัยขึ้นเวลาโดน 
Azure เรียกเครื่องคืนไหมครับ? หรืออยากดูการตั้งค่าร่วมกับ **Horizontal Pod Autoscaler (HPA)** บน Spot?
เมื่อคุณใช้งาน **Spot Instances** บน Azure (หรือ Cloud เจ้าไหนก็ตาม) ปัญหาที่ใหญ่ที่สุดคือ "ความไม่แน่นอน" ครับ เพราะ Azure สามารถดึงเครื่องคืนได้ทุกเมื่อ โดยจะให้เวลาเราเตรียมตัวเพียง **30 วินาที** เท่านั้น

**Node Termination Handler (NTH)** จึงทำหน้าที่เป็น "หูทิพย์" ที่คอยฟังความเคลื่อนไหวจาก Azure Metadata Service และสั่งให้ Kubernetes รีบ "ย้ายของ" (Evict Pods) ออกไปก่อนที่ไฟจะดับครับ

-----

## 1\. การทำงานของ Node Termination Handler

โดยปกติเมื่อ Azure จะยึด Node คืน มันจะส่งสัญญาณผ่าน **Scheduled Events** มายังเครื่องนั้นๆ เครื่องมือ NTH จะทำงานตามลูปนี้:

1.  **Detect:** ตรวจพบสัญญาณ `Preempt` หรือ `Terminate` จาก Azure Metadata Service (IMDS).
2.  **Cordon:** สั่ง Mark Node นั้นเป็น `SchedulingDisabled` เพื่อไม่ให้ Pod ใหม่มาลงเพิ่ม.
3.  **Drain:** สั่งไล่ Pod ที่รันอยู่ออกไปยัง Node อื่นอย่างสุภาพ (Graceful Shutdown).
4.  **Terminate:** เมื่อย้ายเสร็จ หรือครบเวลา 30 วินาที Azure ก็จะดึง Node คืนไป.

-----

## 2\. เครื่องมือที่แนะนำสำหรับ Azure (AKS)

### A. Azure Node Termination Handler (Official OSS)

นี่คือโปรเจกต์ Open Source ของ Azure เองที่ออกแบบมาเพื่อจัดการเรื่องนี้โดยเฉพาะครับ

  * **วิธีการทำงาน:** รันเป็น `DaemonSet` ในทุกๆ Node เพื่อคอยยิง API ไปถาม Metadata ของเครื่องตัวเองบ่อยๆ
  * **ข้อดี:** ติดตั้งง่ายผ่าน Helm Chart และรองรับทั้ง Spot Eviction และการทำ Scheduled Maintenance (เช่น การอัปเดต OS)
  * **Link:** [Azure/kubernetes-node-termination-handler](https://www.google.com/search?q=https://github.com/Azure/kubernetes-node-termination-handler)

### B. AKS Managed Spot Termination Handler (ใหม่ล่าสุด)

ปัจจุบัน AKS มีฟีเจอร์แบบ **Managed** ที่คุณไม่ต้องลงอะไรเองเลยครับ (ยังอยู่ในช่วง Preview หรือเริ่มทยอย Rollout)

  * เมื่อคุณสร้าง Spot Node Pool บน AKS รุ่นใหม่ๆ ระบบจะพยายามจัดการ Drain Node ให้โดยอัตโนมัติเมื่อได้รับสัญญาณจาก Azure
  * **เช็กสถานะ:** คุณสามารถดูได้ในส่วนของ `spotConfig` ใน Node Pool สเปกครับ

-----

## 3\. สิ่งที่ขาดไม่ได้: Pod Disruption Budgets (PDB)

ต่อให้คุณมี NTH ที่เก่งแค่ไหน แต่ถ้าคุณไม่ตั้งค่า **PDB** ระบบอาจจะล่มได้ครับ PDB คือการบอก Kubernetes ว่า "ในขณะที่กำลังย้ายเครื่อง ห้ามให้แอปตัวนี้ตายพร้อมกันเกิน X ตัวนะ"

**ตัวอย่าง YAML สำหรับ PDB:**

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-api-pdb
spec:
  minAvailable: 2 # ต้องมีอย่างน้อย 2 ตัวรันอยู่เสมอ ห้าม Drain จนเหลือน้อยกว่านี้
  selector:
    matchLabels:
      app: web-api
```

-----

## 4\. กลยุทธ์การรับมือ (Resilience Strategy)

เพื่อให้ระบบของคุณ "ถึก" บน Spot Instance ผมแนะนำให้ทำ 3 อย่างนี้คู่กันครับ:

| ลำดับ | สิ่งที่ต้องทำ | เหตุผล |
| :--- | :--- | :--- |
| **1** | **Install NTH** | เพื่อให้รู้ตัวล่วงหน้า 30 วินาทีและเริ่มทำการ Drain |
| **2** | **Set PDB** | เพื่อป้องกันไม่ให้ Pod สำคัญดับไปพร้อมกันทั้งหมดในคราวเดียว |
| **3** | **Graceful Shutdown** | ปรับแต่งแอป (SIGTERM handling) ให้ปิดตัวลงอย่างสะอาดภายใน 20-25 วินาที |

-----

### **คำแนะนำเพิ่มเติม**

หากแอปของคุณใช้เวลานานในการ Shutdown (เช่น ทำงานค้างอยู่และต้องใช้เวลาเคลียร์งานเกิน 30 วินาที) **Spot Instance อาจจะไม่เหมาะครับ** เพราะ Azure จะดึงปลั๊กออกทันทีเมื่อครบ 30 วินาทีโดยไม่สนว่างานจะเสร็จหรือไม่

คุณสนใจให้ผมช่วยเขียน **Helm Chart คาสั่งติดตั้ง Azure Node Termination Handler** หรืออยากดูวิธีเขียนโค้ดในแอปเพื่อรับสัญญาณ **SIGTERM** ให้ทำงานได้เนียนขึ้นไหมครับ?

ขอดูโคดตัวอย่าง **SIGTERM** --

เมื่อ Kubernetes (หรือ Node Termination Handler) สั่งให้ Pod ของคุณปิดตัวลง มันจะส่งสัญญาณที่เรียกว่า **SIGTERM (Signal 15)** ไปยัง Process หลัก (PID 1) ของแอปพลิเคชันครับ

หากแอปของคุณไม่ดักจับสัญญาณนี้ มันจะถูก "ฆ่า" ทันที ซึ่งอาจทำให้ข้อมูลค้างคาหรือ Database Connection ตัดไปดื้อๆ นี่คือวิธีเขียนโค้ดดักจับในภาษาต่างๆ ครับ

---

## 1. Node.js (Express)
Node.js มีความสามารถในการรับ Event ของ `process` ได้โดยตรงครับ

```javascript
const express = require('express');
const app = express();
const server = app.listen(3000);

// ดักจับสัญญาณ SIGTERM
process.on('SIGTERM', () => {
  console.log('ได้รับสัญญาณ SIGTERM: กำลังปิดเซิร์ฟเวอร์อย่างนุ่มนวล...');
  
  // หยุดรับ Request ใหม่ และเคลียร์งานที่ค้างอยู่
  server.close(() => {
    console.log('เคลียร์ Request หมดแล้ว ปิดการเชื่อมต่อ Database...');
    // ปิด DB connection ตรงนี้
    process.exit(0); // ออกจากโปรแกรมด้วยสถานะปกติ
  });

  // ถ้าผ่านไป 20 วินาทียังปิดไม่ลง ให้บังคับปิดเลย (Azure ให้เวลา 30 วิ)
  setTimeout(() => {
    console.error('ปิดไม่ทันเวลา! บังคับปิด...');
    process.exit(1);
  }, 20000);
});
```

---

## 2. Python (Flask/FastAPI)
ใช้โมดูล `signal` ซึ่งเป็นมาตรฐานของ Python ครับ

```python
import signal
import sys
import time

def graceful_shutdown(signum, frame):
    print(f"ได้รับสัญญาณ {signum}: กำลังเคลียร์งาน...")
    # ใส่ Logic การปิดไฟล์ หรือปิดการเชื่อมต่อ DB ที่นี่
    time.sleep(2) # สมมติว่ากำลังเคลียร์งาน
    print("ทำความสะอาดเสร็จสิ้น บาย!")
    sys.exit(0)

# ลงทะเบียนสัญญาณ SIGTERM
signal.signal(signal.SIGTERM, graceful_shutdown)

print("แอปพลิเคชันกำลังทำงาน...")
# ส่วนของ Logic แอปหลัก
```

---

## 3. Go (Golang)
Go ใช้ `channel` ในการรับสัญญาณครับ เป็นวิธีที่ทรงพลังมาก

```go
package main

import (
	"fmt"
	"os"
	"os/signal"
	"syscall"
	"time"
)

func main() {
	// สร้าง channel เพื่อรอรับสัญญาณ
	sigs := make(chan os.Signal, 1)
	// ระบุว่าต้องการดัก SIGTERM
	signal.Notify(sigs, syscall.SIGTERM, syscall.SIGINT)

	go func() {
		sig := <-sigs
		fmt.Printf("\nได้รับสัญญาณ: %s\n", sig)
		// ทำงาน Cleanup ที่นี่
		fmt.Println("กำลังปิดระบบ...")
		os.Exit(0)
	}()

	fmt.Println("โปรแกรมกำลังทำงาน... กด Ctrl+C หรือส่ง SIGTERM เพื่อทดสอบ")
	select {} // ปล่อยให้โปรแกรมรันค้างไว้
}
```

---

## 4. Java (Spring Boot)
ใน Spring Boot เราสามารถใช้ `@PreDestroy` หรือ `ShutdownHook` ได้ครับ

```java
import javax.annotation.PreDestroy;
import org.springframework.stereotype.Component;

@Component
public class MyShutdownHandler {

    @PreDestroy
    public void onShutdown() {
        System.out.println("ได้รับสัญญาณ Shutdown: กำลังปิดการเชื่อมต่อทรัพยากร...");
        // Logic การปิดแอปจะอยู่ที่นี่อัตโนมัติเมื่อได้รับ SIGTERM
    }
}
```

---

## ข้อควรจำในฝั่ง Kubernetes (YAML)

เพื่อให้โค้ดข้างต้นมีเวลาทำงานจนเสร็จ คุณต้องตั้งค่า `terminationGracePeriodSeconds` ในไฟล์ YAML ของคุณให้สัมพันธ์กับเวลาที่แอปต้องใช้ (และต้องไม่เกินเวลาที่ Azure ให้)

```yaml
spec:
  containers:
  - name: my-app
    image: my-app:v1
  # โดยปกติค่าเริ่มต้นคือ 30 วินาที ซึ่งพอดีกับ Azure Spot
  # แต่ถ้าแอปคุณปิดช้ามาก ลองปรับให้เหมาะสม
  terminationGracePeriodSeconds: 30 
```

### **คำเตือนที่สำคัญ (Pitfall)**
ถ้าคุณใช้ Shell script เป็น Entrypoint ใน Dockerfile เช่น:
`CMD ["sh", "-c", "node index.js"]`
สัญญาณ SIGTERM จะส่งไปที่ **Shell (sh)** แทนที่จะส่งไปที่ **Node.js** ทำให้แอปไม่ได้รับสัญญาณ! 

**วิธีแก้:** ให้รันแอปโดยตรง 
`CMD ["node", "index.js"]` หรือใช้ `exec` ใน script ครับ

คุณสนใจอยากให้ผมช่วยตรวจ Dockerfile ของคุณไหมครับว่ามันรองรับการส่งผ่านสัญญาณ SIGTERM ได้ถูกต้องหรือยัง?
