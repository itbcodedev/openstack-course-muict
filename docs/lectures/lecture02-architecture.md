---
title: "การบรรยายที่ 2: สถาปัตยกรรมและส่วนประกอบของ OpenStack"
---

# การบรรยายที่ 2: สถาปัตยกรรมและส่วนประกอบของ OpenStack

## วัตถุประสงค์การเรียนรู้

- เข้าใจสถาปัตยกรรมเชิงตรรกะ (Logical Architecture) ของ OpenStack
- ระบุบทบาทของแต่ละส่วนประกอบหลัก
- เรียนรู้เกี่ยวกับการสื่อสารภายใน: APIs, Message Queue และ Database
- เชื่อมโยง Instance, Network Port และบันทึกเหตุการณ์จาก DevStack ด้วย Instance UUID

---

## สถาปัตยกรรมเชิงตรรกะ

OpenStack ใช้สถาปัตยกรรมแบบ Microservices โดยแต่ละบริการ:

- รับผิดชอบงานเฉพาะด้าน เช่น Compute, Networking หรือ Identity
- ประสานงานผ่าน REST API และใช้ RPC ผ่าน Message Queue ในบางขั้นตอนภายในบริการ
- รองรับการกระจายงานตามบทบาทของบริการและรูปแบบการติดตั้ง

### Control Node vs. Compute Node

| Control Node | Compute Node |
|--------------|--------------|
| ทำงาน API services, scheduler, database, message queue | ทำงาน Hypervisor (VMs) |
| Keystone, Glance, Neutron Server, Nova API/Scheduler | Nova Compute, Neutron Agent |
| ปกติใช้ 2+ เครื่องสำหรับ HA | Scale out ตาม Workload |

---

## กระบวนการสื่อสารภายใน

### ตัวอย่าง: การสร้าง VM

ลำดับย่อนี้ใช้กรณีบูตจาก Image บน DevStack ที่ใช้ libvirt/KVM โดยจัดกลุ่มขั้นตอนตามบทบาทของบริการ

1. **รับคำขอ** — ผู้ใช้ส่งคำขอผ่าน Horizon หรือ CLI ไปยัง Nova API โดยใช้ Token จาก Keystone
2. **เลือก Compute Host** — Nova Conductor ประสานงานกับ Nova Scheduler ซึ่งใช้ข้อมูลทรัพยากรจาก Placement ประกอบการเลือกโฮสต์
3. **เตรียมและเริ่ม VM** — Nova ส่งงานไปยัง `nova-compute` บนโฮสต์ที่เลือกผ่าน RPC โดย Glance ให้บริการ Image และ Neutron จัดการพอร์ตเครือข่ายตามคำขอ ส่วน `nova-compute` ใช้ไดรเวอร์ libvirt จัดการ VM บน KVM
4. **รายงานสถานะ** — ผู้ใช้ดูสถานะ Instance ผ่าน Nova API โดยใช้ Horizon หรือ CLI

---

## เทคโนโลยีสำคัญ

### Message Queue: RabbitMQ / AMQP

- บริการต่าง ๆ สื่อสารแบบ Asynchronous ผ่าน Message Broker
- ลดการผูกพัน (Coupling) ระหว่างส่วนประกอบ
- ตัวอย่างใน DevStack: RabbitMQ เป็น Message Broker สำหรับงาน RPC ภายใน Nova

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

## Workshop ท้ายบท: ตามรอยการสร้าง VM บน DevStack

**เวลา:** 45–60 นาที · **รูปแบบ:** กลุ่มละ 2–3 คน

**ก่อนเริ่ม:** ใช้ DevStack ที่ติดตั้งแล้ว ผู้สอนเตรียม Project, Image, Flavor และ Network สำหรับสร้าง VM ผ่าน Horizon พร้อมตั้งค่า OpenStack CLI ให้ชี้ไปยัง Project เดียวกัน ผู้เรียนใช้บัญชีของกลุ่ม ส่วนผู้สอนสาธิตการอ่าน Log บนโฮสต์ DevStack

### ขั้นที่ 1 — สร้าง Instance และบันทึกจุดเริ่มต้น (10 นาที)

1. ใช้ขั้นตอน Launch Instance จาก [Workshop ใน Lecture 01](lecture01-introduction.md) สร้าง VM ชื่อ `lec02-teamXX-vm01` โดยแทน `XX` ด้วยเลขกลุ่ม
2. จดเวลากด Launch พร้อมเขตเวลา และบันทึก Instance UUID จากหน้ารายละเอียดใน Horizon
3. จดสถานะที่สังเกตได้จนถึง `ACTIVE` หรือ `ERROR` หากพบ `ERROR` ให้เก็บข้อความที่แสดงไว้ประกอบการอ่าน Log

### ขั้นที่ 2 — จับคู่ Instance กับ Network Port (10–15 นาที)

แทนค่าในตัวแปรด้วย UUID ของ VM ที่สร้างจริง แล้วใช้คำสั่งอ่านข้อมูลต่อไปนี้ใน Bash

```bash
INSTANCE_ID='แทนด้วย Instance UUID'
openstack server show "$INSTANCE_ID"
openstack port list --server "$INSTANCE_ID"
```

จากรายการพอร์ต เลือก Port ID ที่ตรงกับ Fixed IP ใน Horizon แล้วใช้ `openstack port show PORT_ID` โดยแทน `PORT_ID` ด้วยค่าจริง เปรียบเทียบ `device_id` กับ Instance UUID และบันทึก `fixed_ips` กับ `network_id` หาก VM อยู่ในสถานะ `ERROR` และยังไม่มีพอร์ต ให้บันทึกผลตามที่พบ

### ขั้นที่ 3 — ผู้สอนสาธิตการตามรอยจาก Log (15–20 นาที)

บนโฮสต์ DevStack ที่ใช้ systemd ให้ตรวจชื่อ Unit ที่ติดตั้งจริงก่อน จากนั้นแทนค่า Instance UUID และช่วงเริ่มต้นการทดลองตามเวลาท้องถิ่นของโฮสต์

```bash
systemctl list-units --all --type=service 'devstack@*'

INSTANCE_ID='แทนด้วย Instance UUID'
SINCE='YYYY-MM-DD HH:MM:SS'
sudo journalctl --since "$SINCE" --unit 'devstack@n-*' --no-pager \
  | grep -F -- "$INSTANCE_ID"
```

- อ่านชื่อ Unit และข้อความประกอบเพื่อระบุว่าเหตุการณ์มาจาก Scheduler, Conductor หรือ Compute หาก Log มี Request ID ให้ใช้ค้นต่อได้
- หากค้นด้วย UUID แล้วไม่พบ ให้ตรวจช่วงเวลา ชื่อ Unit และตำแหน่ง Log ของ Nova API ตามการติดตั้งจริง แล้วอ่านเหตุการณ์ในช่วงนั้น ระบุให้ชัดว่าบรรทัดใดจับคู่กับ VM ได้จาก UUID หรือ Request ID และบรรทัดใดตรงกันเพียงช่วงเวลา
- จากรายชื่อ Unit ให้แยกว่าอะไรทำหน้าที่ใน Control Plane และอะไรจัดการ VM บน Compute Host ในการติดตั้ง All-in-One ทั้งสองบทบาทอยู่บนโฮสต์เดียวกัน

### งานส่งและการคืนทรัพยากร (10–15 นาที)

- วาดลำดับจาก Launch Instance ถึงสถานะที่พบ แล้วแนบหลักฐานอย่างน้อย 2 จุด เช่น รายละเอียด Instance จาก CLI และพอร์ตที่มี Fixed IP ตรงกับ Horizon หรือ Log ที่ผูกกับ UUID ได้
- กำกับแต่ละขั้นว่า **พบหลักฐานจริง** หรือ **อธิบายจากสถาปัตยกรรม** เช่น รายละเอียด Instance และ Port ยืนยันการเชื่อมโยงทรัพยากร ส่วนการส่ง RPC และการเขียนฐานข้อมูลต้องมีหลักฐานของขั้นตอนนั้นจึงจะยืนยันได้
- เมื่อเก็บหลักฐานครบ ให้ลบ Instance ที่กลุ่มสร้างผ่าน Horizon และตรวจว่า VM หายจากรายการของ Project

## สไลด์การบรรยาย

📥 [ดาวน์โหลดสไลด์การบรรยายที่ 2 (PDF)](../assets/slides/OpenStack_Nova_Architecture_Blueprint.pdf){ .md-button }

## แหล่งข้อมูลเพิ่มเติม

- [OpenStack Architecture Guide](https://docs.openstack.org/arch-design/)
- [Nova Compute schedulers](https://docs.openstack.org/nova/latest/admin/scheduling.html)
- [DevStack — Using Systemd](https://docs.openstack.org/devstack/latest/systemd.html)
- [Lab 3 — OpenStack CLI](../labs/lab03-cli.md)
- [RabbitMQ Tutorials](https://www.rabbitmq.com/getstarted.html)
