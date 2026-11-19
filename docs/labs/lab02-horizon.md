---
title: "การทดลองที่ 2: Horizon Dashboard และการจัดการ Project"
---

# การทดลองที่ 2: Horizon Dashboard และการจัดการ Project

## วัตถุประสงค์

สำรวจอินเทอร์เฟซเว็บ Horizon และดำเนินการพื้นฐานของ Cloud ผ่าน GUI 

![](./images/horizon_dashboard.png)

[รายละเดียด Project Hozizon official web https://docs.openstack.org/horizon/latest/](https://docs.openstack.org/horizon/latest/)

---

## ส่วนที่ 1: ภาพรวม Horizon 

### lgoin เข้าสู่ระบบ ผ่าน Dashboard

1. เปิดเบราว์เซอร์: `http://10.0.0.11/dashboard`
2. Domain: `default`
3. Username: `admin`  `demo`
4. Password: `Muict2024` (หรือ ADMIN_PASSWORD ที่ตั้งไว้)

- login horizon dashboard
![](./images/lab02-1.png)

- user admin
![](./images/lab02-2.png)
- user admin มี 2 project
![](./images/lab02-1.png)
- user demo
![](./images/lab02-3.png)


### โครงสร้าง openstack API รองรับการเชื่อมต่อ กับ Horizon, Openstack Cli, REST API

![](./images/horizon_api.png)

---

## ส่วนที่ 2: สร้าง Project และ User ผ่าน Horizon (Keystone Service)
### สร้าง Project
- 1 ไปที่ **Identity → Projects → Create Project**
- 2 กรอกข้อมูล:
   - Project Name: `student-lab`
   - Description: `โครงการ Lab รายวิชา OpenStack`
- 3 คลิก **Create Project**
![](./images/horizon_lab1.png)
![](./images/horizon_lab2.png)
![](./images/horizon_lab3.png)
- 4 Verify โดยใช้ Openstack Cli เพื่อ verify 
!!! tip "ทบทวนความรู้ การ login ด้วย command line"
    - ssh student@ip
    - sudo -u stack -i
    - source ~/devstack/openrc admin admin
    - openstack project -h
    - openstack project list

![](./images/horizon_lab4.png)
![](./images/horizon_lab5.png)



### สร้าง User

- 1 ไปที่ **Identity → Users → Create User**
- 2 กรอกข้อมูล:  
      - User Name: `student01`
      - Password: `password123`
      - Primary Project: `student-lab`
      - Role: `member`
- 3 คลิก **Create User**
![](./images/horizon_lab6.png)
![](./images/horizon_lab7.png)
![](./images/horizon_lab8.png)

- 5 Verify โดยใช้ Openstack Cli เพื่อ verify 
![](./images/horizon_lab9.png)
- 6 Login Dashboard ให้ logout จาก Admin
![](./images/horizon_lab10.png)
![](./images/horizon_lab11.png)
- 7 แสดง  user และ project ที่  user สังกัดอยู่
![](./images/horizon_lab12.png)
- 8 เพื่อให้ user student01 สามารถ login ผ่าน command line ได้ ก็ให้ Download file openrc จาก หน้า Horizone เพื่อนนำมาใช้ สำหรับการ Login ผา่น command line
![](./images/horizon_lab13.png)
![](./images/horizon_lab14.png)
- 9 File จะ Download ไปที่ Download ของ user student เปิด ด้วย file browser ใน ubuntu แล้ว กด Click ขวาเพื่อเลือก Terminal
![](./images/horizon_lab15.png)
- 10 การ Login คือการ สร้าง ตัวแปร ใน shell variable ด้วยคำสั่ง Source เหมือนกับ ตอนที่ login ด้วย user admin
![](./images/horizon_lab16.png)
---

## ส่วนที่ 3: สร้าง image ผ่าน Horizon
![](./images/glance_lab1.png)
### ขั้นตอนที่ 1: อัปโหลด Image Glance Service  (image store)

- 1 Download Image Location: `https://github.com/cirros-dev/cirros/releases/download/0.6.3/cirros-0.6.3-x86_64-disk.img`
![](./images/horizon_lab17.png)
```
wget https://github.com/cirros-dev/cirros/releases/download/0.6.3/cirros-0.6.3-x86_64-disk.img
```
- 2 โหลด ใน ubuntu ที่ Folder Download  ส่วน Link เปิด ใน ค้นหาใน ubuntu
![](./images/horizon_lab17-1.png)
- 3 สร้าง image ด้วย การ login ที่ Horiaon ไปที่ **Project → Compute → Images → Create Image**
![](./images/horizon_lab18.png)

- 4 กรอกข้อมูล
    - ตั้งชื่อ cirrors-student-lab
    - เลือก Image Source: File ไปยัง cirros image ที่ Download  มา
    - Image Format: QCOW2 — QEMU Emulator
![](./images/horizon_lab19.png)
![](./images/horizon_lab20.png)
- Verify ผ่าน Command line
![](./images/horizon_lab21.png)

[คำอธิบาย เกี่ยวกับ options สำหรับการสร้าง image ](./lab04-glance.md)

### ขั้นตอนที่ 2: สร้าง Network  (Neutron Service)

- 1 ไปที่ **Project → Network → Networks → Create Network**
- 2 Network Name: `student-net`
- 3 Subnet Name: `student-subnet`
- 4 Network Address: `192.168.100.0/24`
- 5 Gateway IP: `192.168.100.1`
- 6 คลิก **Create**
![](./images/neutron_lab1.png)
![](./images/neutron_lab2.png)
![](./images/neutron_lab3.png)
![](./images/neutron_lab4.png)
![](./images/neutron_lab5.png)
- 7 Click ชื่อ network และ Click ที่ tab subnet
![](./images/neutron_lab6.png)
- 8 Click Tab Port จะเห็น 192.168.100.2 คือ ip ของ DHCP Server
![](./images/neutron_lab7.png)
- 9 Verify ด้วย Command line
```bash
opestack network list
openstack subnet list
openstack port list
```
![](./images/neutron_lab8.png)


### ขั้นตอนที่ 3: สร้าง Router

- 1 **Project → Network → Routers → Create Router**
- 2 Router Name: `student-router`
- 3 External Network: `public`
- 4 คลิก **Create Router**
![](./images/router_lab1.png)
![](./images/router_lab2.png)
- 5 คลิกชื่อ router → **Interfaces → Add Interface**
![](./images/router_lab3.png)
![](./images/router_lab4.png)
- 6 เลือก Subnet: `student-subnet` เพื่อเชื่อมต่อกับ Router 
![](./images/router_lab5.png)
- 7 คลิก **Submit**
![](./images/router_lab6.png)

!!! info
    - ให้สังเกตุ ว่า ip 192.168.100.1 ของ subnet จะเป็น ip ของ interface ของ Router

- 8 เลือก network Topology
![](./images/router_lab7.png)


### ขั้นตอนที่ 4: กำหนดค่า Security Group

- 1 **Project → Network → Security Groups**
- 2 คลิก **Manage Rules** ที่ `default`
- 3 **Add Rule**:
    - Rule: SSH
    - Remote: CIDR
    - CIDR: `0.0.0.0/0`
- 4 เพิ่มอีกอัน:
    - Rule: ICMP (สำหรับ ping)
![](./images/secgroup_lab1.png)
![](./images/secgroup_lab2.png)
![](./images/secgroup_lab3.png)
![](./images/secgroup_lab4.png)
![](./images/secgroup_lab5.png)
- 5 Verify ด้วย Command line `openstack security group ` 
![](./images/secgroup_lab6.png)

### ขั้นตอนที่ 5: สร้าง VM Instance (Nova Service)

- 1 **Project → Compute → Instances → Launch Instance**
- 2 Details:
    - Instance Name: `vm-01`
- 3 Source:
    - Select Boot Source: Image
    - เลือก Image cirros
- 4 Flavor: `m1.tiny`
- 5 Networks: `student-net`
- 6 Security Groups: `default`
- 7 Key Pair: สร้างหรือนำเข้า
- 8 คลิก **Launch Instance**

### ตรวจสอบ
```bash
openstack image list
openstack flavor list
```
![](./images/compute_lab21.png)



![](./images/compute_lab1.png)
![](./images/compute_lab2.png)
- กดลูกศรชึ้นเป็นการเลือก image มาใช้งานม เลือก Create new Volume จะไปขอ Disk จาก Cinder มาใช้ แบบถาวร แต่ต้องระบุ ขนาด Disk ในที่นี้เลือก 1GB เพราะเป็น Cirros
![](./images/compute_lab3.png)
- กด next เพื่อเข้าสู่ การตั้ง ค่า Flavor
![](./images/compute_lab4.png)
![](./images/compute_lab5.png)
- กด next เพื่อเข้าสู่ การตั้งค่า 
![](./images/compute_lab6.png)
- openstack เลือก Student-net ให้เอง จริงแล้วถึงตรงนี้เรา ก็สามารถ กด launch instance ได้แล้ว แต่เราจะไปตั้งค่าส่วนต่อไป อย่าพึงกดสร้าง 
![](./images/compute_lab7.png)
- กด next
![](./images/compute_lab8.png)
- กด next
![](./images/compute_lab9.png)

!!! info
    หยุดอ่าน   
    - ขั้นตอนต่อไปเป็นการสร้าง Keypair ไว้สำหรับการ ssh ไปยัง vm ร่วมกับการใช้งาน password  
    - `Create Key pair` เป็นการสร้าง key pair, โดย แบ่งเป็น private key และ Public key  
    - openstack จะทำการ inject public key ไปยัง VM ที่เราสร้าง และให้เรา Download/ Copy private มาเก็บไว้  
    - key ไม่สามารถสร้าง ใหม่ หรือย้อนกลับไปดูได้ ดังนัน ต้อง Save ไว้  
    - ใน lab ให้ save ไว้ ที่ /home/student/download/ my_private_key.key 
    - สร้างเป็น ชนิด ssh  
   
- กด `Create key pair`
![](./images/compute_lab10.png)
- ตั้งชือ `my_private_key` และ เลือก ชนิด `ssh`
![](./images/compute_lab11.png)
- กด copy Privatekey ก่อน แล้ว จึงค่อยกด Done (หากลืม กด save สามารถไป ลบ ได้ ที่ Project > Compute > keypair)
![](./images/compute_lab12.png)

- กด launch instance 
![](./images/compute_lab16.png)
![](./images/compute_lab17.png)
![](./images/compute_lab18.png)

## วิธี save private key ใน Ubuntu โดยใช้ `text` หรือ ถ้าฉนัดใช้ Vim ก็ได้
- เปิด Programe ชื่อ `text` ใน Ubuntu ด้วย การ ค้นหาคำว่า `text` แล้ว กด Ctrl+v เพื่อ paste ค่า  private key จาก clipboard แล้วกด Save ชื่อ my_private_key.key ใน /home/student/Download
![](./images/compute_lab13.png)
![](./images/compute_lab14.png)
![](./images/compute_lab15.png)

- ดู volume (cinder service)
![](./images/compute_lab19.png)

- ดู Network Topology
![](./images/compute_lab20.png)

!!! note
    - Volume Size (GB) คือ ขนาดความจุของฮาร์ดดิสก์เสมือน (Persistent Block Storage / Cinder Volume) ที่ OpenStack จะสร้างขึ้นมาใหม่เพื่อใช้เป็นดิสก์หลัก (Root Disk) สำหรับบูตและเก็บข้อมูลของ Instance (VM) 
    - สำหรับ CirrOS (ขนาด ~20 MB) ตั้งค่า 1 GB สามารถใช้งานได้สบาย แต่หากเปลี่ยนไปใช้ Ubuntu Server หรือ OS อื่นๆ ควรปรับเพิ่มเป็น 10–20 GB ขึ้นไปครับ
    - เลือก Volume size , nova จะไปร้องขอ Disk จาก Cinders
---

## ส่วนที่ 4: จัดสรรและผูก Floating IP

- 1 **Project → Network → Floating IPs → Allocate IP**
- 2 Pool: `public`
- 3 คลิก **Allocate IP**
- 4 คลิก **Associate** → เลือก `vm-01`
- 5 คลิก **Associate**

![](./images/floating_lab1.png)
- กด `+` เพื่อ ดึง ค่า floating ip จาก pool แล้วกด `Associate`

![](./images/floating_lab2.png)
![](./images/floating_lab3.png)
- กด Associate ip ที่ได้จะอยู่ใน subnet 192.168.56.192/24
![](./images/floating_lab4.png)
- ดู ผลที่ได้ Project > Network > Floating IPS เมนู
![](./images/floating_lab5.png)

- การตรวจสอบ 
```
openstack server list          # ยืนยันว่า cli-vm-01 อยู่ในสถานะ ACTIVE
openstack floating ip list     # ยืนยันการผูก
openstack volume list          # ยืนยันการเชื่อมต่อวอลุ่ม
```
![](./images/floating_lab6.png)




## End of Week 2 here





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
