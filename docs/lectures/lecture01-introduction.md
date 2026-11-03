# การบรรยายที่ 1: บทนำสู่ Cloud Computing และ OpenStack

## วัตถุประสงค์การเรียนรู้

- เข้าใจรูปแบบบริการและรูปแบบการติดตั้งของ Cloud Computing
- รู้จักประวัติและระบบนิเวศของ OpenStack
- ระบุบริการหลักของ OpenStack และบทบาทของแต่ละบริการ

---

## Cloud Computing คืออะไร?

Cloud Computing คือการส่งมอบทรัพยากรไอทีตามความต้องการผ่านอินเทอร์เน็ต โดยมีการคิดค่าบริการแบบ Pay-as-you-go

### รูปแบบบริการ (Service Models)

| รูปแบบ | คำอธิบาย | ตัวอย่าง |
|-------|-------------|---------|
| **IaaS** | Infrastructure as a Service — เช่า VM, Storage, Network | OpenStack, AWS EC2 |
| **PaaS** | Platform as a Service — Deploy แอปโดยไม่จัดการ Infrastructure | Heroku, OpenShift |
| **SaaS** | Software as a Service — ใช้แอปผ่านเว็บ | Gmail, Salesforce |

### รูปแบบการติดตั้ง (Deployment Models)

- **Public Cloud** — Infrastructure แชร์กันใช้ (AWS, Azure, GCP)
- **Private Cloud** — Infrastructure สำหรับองค์กรเดียว
- **Hybrid Cloud** — ผสมผสาน Public และ Private
- **Community Cloud** — แชร์ระหว่างองค์กรที่มีเป้าหมายร่วมกัน

---

## ภาพรวม OpenStack

!!! quote "พันธกิจของ OpenStack"
    สร้างแพลตฟอร์ม Cloud Computing แบบ Open Source ที่เป็นที่นิยมใช้ทั้งในภาครัฐและเอกชน โดยไม่คำนึงถึงขนาด

### ประวัติ

- **2010** — โครงการร่วมระหว่าง NASA และ Rackspace
- **2012** — ก่อตั้ง OpenStack Foundation
- **2023** — OpenInfra Foundation ควบคุมโครงการ
- **ล่าสุด** — Antelope (2023.1), Bobcat (2023.2), Caracal (2024.1)

### ทำไมต้องใช้ OpenStack?

✅ Open Source — ไม่ผูกขาด Vendor  
✅ Ecosystem บริการกว้างขวาง  
✅ ใช้งานในหลายพันองค์กรทั่วโลก  
✅ Community สนับสนุนเข้มแข็ง  
✅ API รองรับความเข้ากันได้กับ AWS (ในบริการหลายตัว)

---

## ภาพรวมบริการหลัก

| บริการ | ชื่อโค้ด | หน้าที่ |
|---------|----------|---------|
| Identity | **Keystone** | ยืนยันตัวตนและสิทธิ์ |
| Compute | **Nova** | จัดเตรียม Virtual Machine |
| Networking | **Neutron** | บริการเชื่อมต่อเครือข่าย |
| Image | **Glance** | จัดเก็บและจัดการรูป VM |
| Block Storage | **Cinder** | พื้นที่จัดเก็บ Block แบบถาวร |
| Object Storage | **Swift** | ระบบจัดเก็บ Object แบบขยายได้ |
| Dashboard | **Horizon** | หน้า Web UI |
| Orchestration | **Heat** | จัดสรรทรัพยากรด้วย Template |
| Telemetry | **Ceilometer** | เก็บข้อมูล Meter และ Monitoring |

---

## ภาพรวมสถาปัตยกรรม

```
┌─────────────────────────────────────────────────────────┐
│                    Horizon (Web UI)                      │
├─────────────────────────────────────────────────────────┤
│  OpenStack REST APIs  │  Python SDKs  │  CLI Clients    │
├─────────────────────────────────────────────────────────┤
│  Nova  │  Neutron  │  Cinder  │  Glance  │  Swift      │
├─────────────────────────────────────────────────────────┤
│  Message Queue (RabbitMQ)  │  Database (MariaDB)        │
├─────────────────────────────────────────────────────────┤
│  Hypervisor (KVM/QEMU)  │  Linux Bridge / OVS           │
└─────────────────────────────────────────────────────────┘
```

---

## สไลด์การบรรยาย

📥 [ดาวน์โหลดสไลด์การบรรยายที่ 1 (PDF)](../assets/slides/lecture01.pdf){ .md-button }

## แหล่งข้อมูลเพิ่มเติม

- [OpenStack Documentation](https://docs.openstack.org/)
- [OpenInfra Foundation](https://openinfra.org/)
- [คำจำกัดความ Cloud Computing ของ NIST (SP 800-145)](https://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-145.pdf)
