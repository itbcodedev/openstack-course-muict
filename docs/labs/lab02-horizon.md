---
title: "การทดลองที่ 2: Horizon Dashboard และการจัดการ Project"
---

# การทดลองที่ 2: Horizon Dashboard และการจัดการ Project

## วัตถุประสงค์

สำรวจอินเทอร์เฟซเว็บ Horizon และดำเนินการพื้นฐานของ Cloud ผ่าน GUI

---

## ส่วนที่ 1: ภาพรวม Horizon

### เข้าสู่ระบบ

1. เปิดเบราว์เซอร์: `http://10.20.30.11/dashboard`
2. Domain: `default`
3. Username: `admin`
4. Password: `Muict@2024` (หรือ ADMIN_PASSWORD ที่ตั้งไว้)

### โครงสร้าง Dashboard

```
┌──────────────────────────────────────────────────────┐
│  Project | Admin | Identity | Settings               │
├──────────────────┬───────────────────────────────────┤
│                  │                                   │
│  Overview        │   ╔═══════════╗                  │
│  Instances       │   ║ Instances ║  0               │
│  Volumes         │   ╚═══════════╝                  │
│  Networks        │                                   │
│  Images          │   ╔═══════════╗                  │
│  Security Groups │   ║  Volumes  ║  0               │
│  Network Topology│   ╚═══════════╝                  │
│                  │                                   │
└──────────────────┴───────────────────────────────────┘
```

---

## ส่วนที่ 2: สร้าง Project และ User

### ผ่าน Horizon

1. ไปที่ **Identity → Projects → Create Project**
2. กรอกข้อมูล:
   - Project Name: `student-lab`
   - Description: `โครงการ Lab รายวิชา OpenStack`
3. คลิก **Create Project**

### สร้าง User

1. ไปที่ **Identity → Users → Create User**
2. กรอกข้อมูล:
   - User Name: `student01`
   - Password: `password123`
   - Primary Project: `student-lab`
   - Role: `member`
3. คลิก **Create User**

---

## ส่วนที่ 3: สร้าง Instance ผ่าน Horizon

### ขั้นตอนที่ 1: อัปโหลด Image

1. ไปที่ **Project → Compute → Images → Create Image**
2. เลือก Image Source: URL
3. Image Location: `https://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img`
4. Image Format: QCOW2 — QEMU Emulator
5. คลิก **Create Image**

### ขั้นตอนที่ 2: สร้าง Network

1. ไปที่ **Project → Network → Networks → Create Network**
2. Network Name: `student-net`
3. Subnet Name: `student-subnet`
4. Network Address: `192.168.100.0/24`
5. Gateway IP: `192.168.100.1`
6. คลิก **Create**

### ขั้นตอนที่ 3: สร้าง Router

1. **Project → Network → Routers → Create Router**
2. Router Name: `student-router`
3. External Network: `public`
4. คลิก **Create Router**
5. คลิกชื่อ router → **Interfaces → Add Interface**
6. Subnet: `student-subnet`
7. คลิก **Submit**

### ขั้นตอนที่ 4: กำหนดค่า Security Group

1. **Project → Network → Security Groups**
2. คลิก **Manage Rules** ที่ `default`
3. **Add Rule**:
   - Rule: SSH
   - Remote: CIDR
   - CIDR: `0.0.0.0/0`
4. เพิ่มอีกอัน:
   - Rule: ICMP (สำหรับ ping)

### ขั้นตอนที่ 5: สร้าง Instance

1. **Project → Compute → Instances → Launch Instance**
2. Details:
   - Instance Name: `vm-01`
3. Source:
   - Select Boot Source: Image
   - เลือก Image cirros
4. Flavor: `m1.tiny`
5. Networks: `student-net`
6. Security Groups: `default`
7. Key Pair: สร้างหรือนำเข้า
8. คลิก **Launch Instance**

---

## ส่วนที่ 4: จัดสรรและผูก Floating IP

1. **Project → Network → Floating IPs → Allocate IP**
2. Pool: `public`
3. คลิก **Allocate IP**
4. คลิก **Associate** → เลือก `vm-01`
5. คลิก **Associate**

!!! success ":white_check_mark: ทดสอบการเชื่อมต่อ"
    จากเครื่อง Host (Windows):
    ```bash
    ping <FLOATING_IP>
    ssh cirros@<FLOATING_IP>
    # Password: gocubsgo
    ```

---

## ส่วนที่ 5: สร้าง Volume และ Attach

1. **Project → Volumes → Volumes → Create Volume**
   - Volume Name: `data-vol`
   - Size: 2 GB
2. คลิก **Create Volume**
3. คลิก **Manage Attachments** → Attach กับ `vm-01`
4. SSH เข้า VM และตรวจสอบ:
   ```bash
   ssh cirros@<FLOATING_IP>
   lsblk
   sudo mkfs.ext4 /dev/vdb
   sudo mkdir /mnt/data
   sudo mount /dev/vdb /mnt/data
   df -h
   ```

---

!!! homework ":clipboard: การบ้าน"
    📋 **รายงานการทดลอง:**

    1. Screenshot หน้า Project Overview
    2. Screenshot แสดง `student-net` network topology
    3. Screenshot `vm-01` กำลังทำงานพร้อม Floating IP
    4. Screenshot SSH session ภายใน VM
    5. ตอบคำถาม: Private network ต่างจาก Public (external) network อย่างไร?
    6. ตอบคำถาม: ทำไมต้องมี router เชื่อมระหว่างทั้งสอง?
