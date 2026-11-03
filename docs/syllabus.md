# 📋 หลักสูตรรายวิชา

## วิชา OpenStack Cloud Computing — MUICT

---

## คำอธิบายรายวิชา

รายวิชานี้กล่าวถึงสถาปัตยกรรม การติดตั้ง และการบริหารจัดการโครงสร้างพื้นฐาน Cloud ด้วย OpenStack นักศึกษาจะได้รับทักษะการปฏิบัติจริงผ่านการทดลอง Lab โดยใช้ DevStack และการติดตั้งระดับ Production

## ผลลัพธ์การเรียนรู้

เมื่อจบรายวิชานี้ นักศึกษาจะสามารถ:

1. อธิบายรูปแบบของ Cloud Computing (IaaS, PaaS, SaaS) และบทบาทของ OpenStack
2. ติดตั้งสภาพแวดล้อม OpenStack ที่ใช้งานได้จริงโดยใช้ DevStack
3. กำหนดค่าและบริหารจัดการบริการหลักของ OpenStack (Keystone, Nova, Neutron, Glance, Cinder, Swift)
4. ออกแบบโทโปโลยีเครือข่ายแบบ Multi-tenant ที่ปลอดภัย
5. Automate การจัดสรรทรัพยากร Cloud ด้วย Heat Templates
6. ตรวจสอบประสิทธิภาพและแก้ไขปัญหาโครงสร้างพื้นฐาน Cloud
7. วางแผนและดำเนินการติดตั้ง OpenStack ระดับ Production

---

## การประเมินผล

| องค์ประกอบ | น้ำหนัก | รายละเอียด |
|-----------|--------|------------|
| รายงาน Lab | 40% | รายงานการทดลองปฏิบัติรายสัปดาห์ |
| โครงงานกลางภาค | 20% | Deploy แอปพลิเคชัน 3-tier บน OpenStack |
| สอบปลายภาค | 20% | ทฤษฎี + ปฏิบัติการแก้ไขปัญหา |
| การเข้าร่วมในชั้นเรียน | 10% | แบบทดสอบ การอภิปราย Peer Review |
| โครงงานปลายภาค | 10% | โครงงานกลุ่ม: ติดตั้ง OpenStack แบบกำหนดเอง |

---

## กำหนดการเรียนการสอน

| สัปดาห์ | หัวข้อ | การทดลอง |
|------|-------|-----|
| 1 | พื้นฐาน Cloud Computing | ติดตั้ง CentOS Stream 10 บน Hyper-V |
| 2 | สถาปัตยกรรมและประวัติ OpenStack | ติดตั้ง DevStack |
| 3 | Keystone — ตัวตนและสิทธิ์ | จัดการ User/Project/Role |
| 4 | Nova — บริการ Compute | สร้างและจัดการ VM |
| 5 | Neutron — เครือข่าย | VLAN, Router, Security Groups |
| 6 | Glance — บริการ Image | สร้าง Image แบบกำหนดเอง |
| 7 | Cinder — Block Storage | จัดการ Volume |
| 8 | Swift — Object Storage | การใช้งาน Object Store |
| 9 | Heat — Orchestration | Deploy ด้วย Template |
| 10 | Ceilometer / Gnocchi — การตรวจสอบ | Metrics และ Alarms |
| 11 | การติดตั้งระดับ Production (Kolla-Ansible) | OpenStack แบบ Containerized |
| 12 | การแก้ไขปัญหาและแนวปฏิบัติที่ดี | Scenarios การ Debug |
| 13 | นำเสนอโครงงานนักศึกษา | — |
| 14 | สอบปลายภาค | — |

---

## เงื่อนไขเบื้องต้น

- เชี่ยวชาญการใช้งาน Command Line ของ Linux
- เครือข่ายเบื้องต้น (TCP/IP, VLANs, Routing)
- คุ้นเคยกับการเขียน Python Script
- แนวคิด Virtualization (KVM, QEMU)

## เครื่องมือที่ต้องใช้

- คอมพิวเตอร์แล็ปท็อปพร้อม Hyper-V และ RAM 16 GB+
- Virtual Machine CentOS Stream 10 (RAM 8 GB, Disk 60 GB)
- SSH Client และ Terminal Access
- บัญชี GitHub สำหรับส่งงาน

## สภาพแวดล้อมที่ใช้ในรายวิชา

| สภาพแวดล้อม | ผลิตภัณฑ์ |
|-----------|---------|
| Host OS | Windows 10/11 Pro พร้อม Hyper-V |
| Guest OS | CentOS Stream 10 |
| Cloud Platform | OpenStack (DevStack) |
| Virtualization | KVM (Nested ใน Hyper-V) |
| Networking | Neutron with Open vSwitch |
