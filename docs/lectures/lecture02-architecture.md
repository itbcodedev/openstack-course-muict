# การบรรยายที่ 2: สถาปัตยกรรมและส่วนประกอบของ OpenStack

## วัตถุประสงค์การเรียนรู้

- เข้าใจสถาปัตยกรรมเชิงตรรกะ (Logical Architecture) ของ OpenStack
- ระบุบทบาทของแต่ละส่วนประกอบหลัก
- เรียนรู้เกี่ยวกับการสื่อสารภายใน: APIs, Message Queue และ Database

---

## สถาปัตยกรรมเชิงตรรกะ

OpenStack ใช้สถาปัตยกรรมแบบ Microservices โดยแต่ละบริการ:

- ทำงานอิสระจากกัน
- สื่อสารผ่าน REST APIs
- สามารถ Scale แบบ Horizontal ได้

### Control Node vs. Compute Node

| Control Node | Compute Node |
|--------------|--------------|
| ทำงาน API services, scheduler, database, message queue | ทำงาน Hypervisor (VMs) |
| Keystone, Glance, Neutron Server, Nova API/Scheduler | Nova Compute, Neutron Agent |
| ปกติใช้ 2+ เครื่องสำหรับ HA | Scale out ตาม Workload |

---

## กระบวนการสื่อสารภายใน

### ตัวอย่าง: การสร้าง VM

```
User → Horizon / CLI
  ↓
Nova API → Keystone (verify token)
  ↓
Nova Scheduler (เลือก compute host)
  ↓
Message Queue → Nova Compute บน host ที่เลือก
  ↓
Glance (ดึง image) + Neutron (จัดสรร network)
  ↓
Libvirt/KVM สร้าง VM บน hypervisor
```

---

## เทคโนโลยีสำคัญ

### Message Queue: RabbitMQ / AMQP

- บริการต่าง ๆ สื่อสารแบบ Asynchronous ผ่าน Message Broker
- ลดการผูกพัน (Coupling) ระหว่างส่วนประกอบ
- ค่าเริ่มต้น: RabbitMQ รองรับ Qpid และ ZeroMQ

### Database: MariaDB / MySQL

- เก็บสถานะถาวรของทุกบริการ
- แต่ละบริการมี Database Schema เป็นของตัวเอง
- ใช้ Galera Cluster สำหรับระบบ HA

### WSGI Server

- Web Services ของ Python ใช้ WSGI (เช่น Apache mod_wsgi, uWSGI)
- รับ HTTP Requests และส่งต่อไปยัง Service Code

---

## ส่วนประกอบของแต่ละบริการ

### Nova (Compute)

| ส่วนประกอบ | หน้าที่ |
|-----------|------|
| nova-api | รับและส่งต่อ API requests |
| nova-scheduler | ตัดสินใจเลือก host ที่จะรัน VM |
| nova-compute | จัดการ VMs บน hypervisor |
| nova-conductor | ตัวกลางเข้าถึง Database |
| nova-console | VNC/SPICE Console Proxy |

### Neutron (Networking)

| ส่วนประกอบ | หน้าที่ |
|-----------|------|
| neutron-server | API และ Plugin Manager |
| L2 Agent (OVS/Linux Bridge) | จัดการ Virtual Switches |
| L3 Agent | จัดการ Routing และ NAT |
| DHCP Agent | จ่าย IP Addresses |
| Metadata Agent | จัดหา Cloud-init Data |

---

## รูปแบบการติดตั้ง (Deployment Topologies)

### All-in-One (DevStack)
- Node เดียว — ทุกอย่างอยู่บนเครื่องเดียว
- เหมาะสำหรับการเรียนรู้และพัฒนา

### Two-Node
- Control node + Compute node
- ขั้นต่ำที่คล้าย Production

### Multi-Node HA
- Controller หลายตัวพร้อม keepalived/haproxy
- Compute nodes หลายตัว
- Storage และ Network nodes แยก

---

## สไลด์การบรรยาย

📥 [ดาวน์โหลดสไลด์การบรรยายที่ 2 (PDF)](../assets/slides/OpenStack_Nova_Architecture_Blueprint.pdf){ .md-button }

## แหล่งข้อมูลเพิ่มเติม

- [OpenStack Architecture Guide](https://docs.openstack.org/arch-design/)
- [RabbitMQ Tutorials](https://www.rabbitmq.com/getstarted.html)
