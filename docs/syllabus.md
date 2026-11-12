---
title: "📋 หลักสูตรรายวิชา"
---

# 📋 หลักสูตรรายวิชา

## วิชา OpenStack Cloud Computing — MUICT

---

## คำอธิบายรายวิชา

ITDS321_Cloud Computing Systems and Applications   
รายวิชานี้กล่าวถึงสถาปัตยกรรม การติดตั้ง และการบริหารจัดการโครงสร้างพื้นฐาน Cloud ด้วย OpenStack 

ITDS322_Cloud Computing Lab   
นักศึกษาจะได้รับทักษะการปฏิบัติจริงผ่านการทดลอง Lab โดยใช้ DevStack และการติดตั้งระดับ Production

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

| องค์ประกอบ | คะแนนดิบ | รายละเอียด |
|-----------|--------|------------|
| ITDS321 | 5 | เก็บคะแนน รายงาน จำนวน 3 ครั้ง |
|  | 10 | ข้อสอบ multiple choice สอบในระบบ FinX |
| ITDS322 | 15 | ไม่มีการสอบ มีแต่ การเก็บคะแนน ใน รายงานการทดลองปฏิบัติรายสัปดาห์ (workshop report)  3 ครั้ง ครั้งละ 5 คะแนน ไม่เก็บคะแนนย้อนหลัง |


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
| 8 option | Swift — Object Storage | การใช้งาน Object Store|
| 9 option | Heat — Orchestration | Deploy ด้วย Template |
| 10 option | Ceilometer / Gnocchi — การตรวจสอบ | Metrics และ Alarms |

---

## เงื่อนไขเบื้องต้น

- เข้าใจ Linux command line 
- เข้าใจ Basic networking (TCP/IP, VLANs, routing) ,  Switch Layer 2, Switch Layer 3
- ความรู้โปรแกรมภาษา Python scripting
- หลักการทำงาน และ การสร้าง Virtualization ของ Hyper V
- การใช้งานคำสั่ง Git เบื่องต้น

## สภาพแวดล้อมที่ใช้ในรายวิชา

| สภาพแวดล้อม | ผลิตภัณฑ์ |
|-----------|---------|
| Host OS | Windows 10/11 Pro พร้อม Hyper-V |
| Guest OS | ubuntu 24.04 |
| Cloud Platform | OpenStack (DevStack) |
| Virtualization | KVM (Nested ใน Hyper-V) |
| Networking | Neutron with Open vSwitch |


ตัวเลือกข้อความสำหรับนำไปใส่ในประมวลรายวิชา (Course Syllabus) หรือสไลด์แนะนำรายวิชา โดยปรับแก้คำผิด (เช่น *อนุญาต* ไม่ใส่สระอิ) และแบ่งตามระดับความเป็นทางการให้เลือกใช้งานได้ทันทีครับ


**แนวทางการใช้ AI Tools ในรายวิชา**

* **Self-Study & Research:** อนุญาตให้ใช้ AI ช่วยค้นคว้าข้อมูลเชิงลึก อธิบายหลักการทำงานของระบบ และยกตัวอย่างประกอบการเรียนรู้
* **Assignments & Reports:** อนุญาตให้ใช้ AI ช่วยในการทำรายงาน ตอบคำถามท้ายบท หรือช่วยวิเคราะห์ปัญหา (Troubleshooting) ได้ แต่ผู้เรียนต้องสามารถอธิบายที่มาและหลักการทำงานของคำตอบนั้นได้
* **Examinations:** **ไม่อนุญาต** ให้ใช้งาน AI Assistance ทุกประเภทในระหว่างการสอบ เพื่อประเมินทักษะและการแก้ปัญหาด้วยตนเองของผู้เรียน
---
