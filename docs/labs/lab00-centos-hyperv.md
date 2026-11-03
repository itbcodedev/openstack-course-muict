# การทดลองที่ 0: ติดตั้ง CentOS Stream 10 บน Hyper-V

## วัตถุประสงค์

ติดตั้งระบบปฏิบัติการ CentOS Stream 10 ลงบน Virtual Machine โดยใช้ Hyper-V บน Windows เพื่อเป็นเครื่องมือสำหรับติดตั้ง OpenStack ด้วย DevStack ในภายหลัง

---

## ข้อกำหนดของเครื่องวิทยุกรรม (Hyper-V Host)

| รายการ | ข้อกำหนด |
|--------|-----------|
| ระบบปฏิบัติการ | Windows 10/11 Pro หรือ Windows Server 2019/2022 |
| เปิดใช้งาน Hyper-V | Enabled ใน BIOS/UEFI และระบบ |
| หน่วยความจำ (RAM) | แนะนำ 16 GB ขึ้นไป |
| พื้นที่ดิสก์ | มีพื้นที่ว่าง 60 GB ขึ้นไป |
| ไฟล์ ISO | CentOS-Stream-10-latest-x86_64-dvd1.iso |

---

## สร้าง Virtual Switch บน Hyper-V

ก่อนสร้าง VM ให้สร้าง Virtual Switch ทั้งหมด 3 ตัวบน Hyper-V Manager:

### 1. Default Switch (เชื่อมต่ออินเทอร์เน็ต)

Hyper-V สร้าง Default Switch มาให้อัตโนมัติ หากไม่มี ให้สร้าง External Switch ที่เชื่อมต่อกับการ์ดเครือข่ายหลัก:

```
Virtual Switch Manager → New virtual network switch → External → Create Virtual Switch
ชื่อ: DefaultSwitch (หรือ External)
Connection type: External network → [เลือกการ์ดเครือข่ายหลัก]
☑ Allow management operating system to share this network adapter
```

![ขั้นตอนสร้าง Default Switch](images/lab00-1.png)

### 2. Internal Management Network

```
Virtual Switch Manager → New virtual network switch → Internal → Create Virtual Switch
ชื่อ: Internal-Mgmt
Connection type: Internal network
```

- ใช้สำหรับเข้าถึงหน้า Web OpenStack (Horizon Dashboard) และ API ของ OpenStack
- IP ภายในเครือข่าย เช่น 10.20.30.0/24

![ขั้นตอนสร้าง Internal-Mgmt Switch](images/lab00-2.png)

### 3. Internal External Network (สำหรับ br-ex ของ Neutron Router)

```
Virtual Switch Manager → New virtual network switch → Internal → Create Virtual Switch
ชื่อ: Internal-BrEx
Connection type: Internal network
```

- ใช้เป็น physical interface สำหรับ Neutron br-ex bridge
- จะกำหนดค่า IP ให้กับอินเทอร์เฟซนี้เมื่อติดตั้ง OpenStack ด้วย DevStack
- IP ภายในเครือข่าย เช่น 192.168.56.0/24

![ขั้นตอนสร้าง Internal-BrEx Switch](images/lab00-3.png)

![Virtual Switch Manager แสดงทั้ง 3 Switches](images/lab00-4.png)

!!! config ":wrench: ตรวจสอบ Virtual Switch"
    หลังจากสร้างเสร็จ ให้ตรวจสอบว่ามีทั้งหมด 3 switches:
    - DefaultSwitch (หรือ External)
    - Internal-Mgmt
    - Internal-BrEx

---

## สร้าง Virtual Machine

### ขั้นตอนที่ 1: New Virtual Machine Wizard

```
Hyper-V Manager → Action → New → Virtual Machine
```

| ขั้นตอน | ค่าที่ตั้ง |
|---------|-----------|
| Name | OpenStack-Controller |
| Generation | Generation 2 (รองรับ UEFI) |
| Memory | 8192 MB (8 GB) — ปิด Dynamic memory (คลิกเครื่องหมายถูกออก) |
| Network | เลือก DefaultSwitch ก่อน (จะเพิ่มอีก 2 adapters ในขั้นตอนหลัง) |
| Hard Disk | Create a virtual hard disk: 60 GB (VHDX) |
| Installation Options | Attach ISO: CentOS-Stream-10-latest-x86_64-dvd1.iso |

![สร้าง VM ขั้นตอนที่ 1](images/lab00-5.png)

![เลือก Generation 2](images/lab00-6.png)

![ตั้งค่า RAM 8GB](images/lab00-7.png)

![เลือก Network Adapter](images/lab00-8.png)

![ตั้งค่า Virtual Hard Disk](images/lab00-9.png)

![Attach ISO file](images/lab00-10.png)

!!! warning ":warning: สำคัญ: ปิด Dynamic Memory"
    OpenStack ต้องการหน่วยความจำคงที่ ให้ปิด Dynamic memory แล้วตั้งค่า Startup RAM = 8192 MB

![สรุปการสร้าง VM](images/lab00-11.png)

### ขั้นตอนที่ 2: เพิ่ม Network Adapter

หลังจากสร้าง VM เสร็จ ให้เปิด **Settings** ของ VM และเพิ่ม Network Adapter อีก 2 ตัว:

```
Settings → Add Hardware → Network Adapter → Add
```

- **Network Adapter 1:** DefaultSwitch (มีอยู่แล้ว)
- **Network Adapter 2:** Internal-Mgmt
- **Network Adapter 3:** Internal-BrEx

รวมทั้งหมด 3 Network Adapters

![เพิ่ม Network Adapter ที่ 2](images/lab00-12.png)

![เพิ่ม Network Adapter ที่ 3](images/lab00-13.png)

![VM Settings แสดง 3 Adapters](images/lab00-14.png)

### ขั้นตอนที่ 3: ปรับแต่งเพิ่มเติม

```
Settings:
├── Security
│   └── Secure Boot: ☐ Uncheck (ปิด Secure Boot — จำเป็นสำหรับ Linux บน Gen 2)
├── Processor
│   └── Number of virtual processors: 2 (หรือ 4 หากมี)
├── Integration Services
│   └── ☑ Guest services (ช่วยให้ Copy-Paste ระหว่าง Host กับ VM ได้)
└── Checkpoint File Location
    └── กำหนดพื้นที่จัดเก็บ Snapshot
```

![ปิด Secure Boot](images/lab00-15.png)

![ตั้งค่า Processor](images/lab00-16.png)

---

## ติดตั้ง CentOS Stream 10

### ขั้นตอนที่ 1: เปิดเครื่อง VM

```
Hyper-V Manager → เลือก VM → Start → Connect
```

หากขึ้นหน้าจอสีดำหรือไม่ boot จาก ISO ให้ตรวจสอบ:
```
Settings → Firmware → Boot order: ลาก DVD ขึ้นมาไว้บนสุด
```

![Boot จาก ISO](images/lab00-17.png)

![เลือก Install CentOS Stream 10](images/lab00-18.png)

### ขั้นตอนที่ 2: ติดตั้ง CentOS Stream 10

1. เลือก **Install CentOS Stream 10**

2. เลือกภาษา: **English** (แนะนำ) หรือ Thai

3. **INSTALLATION SUMMARY** — ตั้งค่าตามลำดับ:

#### ✅ Installation Destination

```
Installation Destination → Local Standard Disks: เลือกดิสก์ 60 GB
Storage Configuration: Automatic
→ Done
```

![Installation Destination](images/lab00-19.png)

![เลือกดิสก์](images/lab00-20.png)

#### ✅ Network & Host Name

```
Network & Host Name
├── Host Name: openstack-controller
├── Ethernet (enp0s3)  [DefaultSwitch]
│   └── สลับเป็น ON → Configure → IPv4 → Automatic (DHCP) → Save
├── Ethernet (enp0s4)  [Internal-Mgmt]
│   └── สลับเป็น ON → Configure → IPv4 → Manual
│       ├── Address: 10.20.30.11
│       ├── Netmask: 255.255.255.0
│       └── Save
└── Ethernet (enp0s5)  [Internal-BrEx]
    └── สลับเป็น ON → Configure → IPv4 → Manual
        ├── Address: 192.168.56.11
        ├── Netmask: 255.255.255.0
        └── Save
→ Done
```

!!! config ":gear: จด IP ไว้ใช้ในภายหลัง"
    - enp0s3 (Default): DHCP - รับ IP อัตโนมัติ
    - enp0s4 (Mgmt): 10.20.30.11/24
    - enp0s5 (BrEx): 192.168.56.11/24 — ต้องปิด or ลบ IP นี้ออกหลังติดตั้ง DevStack

![Network & Host Name - รายการอินเทอร์เฟซ](images/lab00-21.png)

![เปิด enp0s3 - DHCP](images/lab00-22.png)

![ตั้งค่า enp0s4 - Manual IP](images/lab00-23.png)

![ตั้งค่า enp0s5 - Manual IP](images/lab00-24.png)

![Network ตั้งค่าเสร็จแล้ว](images/lab00-25.png)

#### ✅ Time & Date

```
Time & Date → Region: Asia → City: Bangkok → Network Time: ON → Done
```

![Time & Date](images/lab00-26.png)

#### ✅ Software Selection

```
Software Selection → Server with GUI → Done
```

![Software Selection](images/lab00-27.png)

![เลือก Server with GUI](images/lab00-28.png)

แนะนำเลือก **Server with GUI** เพื่อความสะดวกในการเรียนรู้ หรือเลือก **Minimal Install** หากต้องการเบา

#### ✅ Root Password

```
Root Password: ตั้งรหัสผ่านสำหรับ root
```

![Root Password](images/lab00-29.png)

#### ✅ User Creation

```
User Creation
├── Full Name: OpenStack User
├── User name: stack
├── Password: ตั้งรหัสผ่าน
├── Confirm password: ใส่ซ้ำ
└── ☑ Make this user administrator
→ Done
```

!!! warning ":lock: แนะนำใช้ชื่อผู้ใช้ stack"
    DevStack ตั้งค่าให้ติดตั้งภายใต้ผู้ใช้ /opt/stack โดยอัตโนมัติ การใช้ชื่อ **stack** จะทำให้สะดวกขึ้น

### ขั้นตอนที่ 3: เริ่มติดตั้ง

```
→ Begin Installation
```

รอประมาณ 10-20 นาทีให้การติดตั้งเสร็จสิ้น จากนั้น:

```
→ Reboot System
```

![Begin Installation](images/lab00-32.png)

![ระหว่างติดตั้ง](images/lab00-33.png)

![ติดตั้งเสร็จแล้ว - Reboot](images/lab00-34.png)

!!! tip ":bulb: ตรวจสอบ ISO หลัง Reboot"
    หลังจาก Reboot ให้ปิด VM → Settings → DVD Drive → ลบ ISO ออก → เปิด VM ใหม่

---

## ตั้งค่าหลังติดตั้ง (Post-Installation)

### ขั้นตอนที่ 1: เข้าสู่ระบบ

```
Username: stack
Password: [รหัสที่ตั้งไว้]
```

![เข้าสู่ระบบ CentOS Stream 10](images/lab00-35.png)

### ขั้นตอนที่ 2: ตรวจสอบ Network Interfaces

เปิด Terminal แล้วรันคำสั่ง:

```bash
ip addr show
```

ควรเห็น 3 interfaces:
- `enp0s3`: มี IP จาก DHCP (เชื่อมต่ออินเทอร์เน็ตได้)
- `enp0s4`: 10.20.30.11 — Management Network
- `enp0s5`: 192.168.56.11 — External Network (สำหรับ br-ex)

![ตรวจสอบ IP ด้วย ip addr](images/lab00-36.png)

### ขั้นตอนที่ 3: อัปเดตระบบ

```bash
sudo dnf update -y
sudo dnf install -y epel-release git vim wget curl net-tools bash-completion
```

![อัปเดตระบบ](images/lab00-37.png)

### ขั้นตอนที่ 4: ปิด Firewall (สำหรับ Lab)

```bash
sudo systemctl disable --now firewalld
```

หรือถ้าต้องการเปิดใช้งานให้เปิดพอร์ตที่จำเป็น:
```bash
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

![ปิด Firewall](images/lab00-38.png)

!!! warning ":no_entry: ห้ามปิด Firewall บน Production"
    ในสภาพแวดล้อมจริงควรปรับแต่ง Firewall policy แทนการปิด

### ขั้นตอนที่ 5: ปิด SELinux (สำหรับ Lab)

```bash
sudo setenforce 0
sudo sed -i 's/SELINUX=enforcing/SELINUX=permissive/g' /etc/selinux/config
```

![ปิด SELinux](images/lab00-39.png)

!!! warning ":no_entry: ห้ามปิด SELinux บน Production"
    ในสภาพแวดล้อมจริงควรปรับแต่ง SELinux policy แทนการปิด

### ขั้นตอนที่ 6: ตั้งค่า sudo สำหรับ stack user

```bash
sudo visudo
```

เพิ่มบรรทัดนี้ลงไป:
```
stack ALL=(ALL) NOPASSWD: ALL
```

![ตั้งค่า sudo สำหรับ stack user](images/lab00-40.png)

### ขั้นตอนที่ 7: สร้าง Snapshot

ก่อนติดตั้ง DevStack ให้สร้าง Checkpoint (Snapshot) ไว้:

```
Hyper-V Manager → เลือก VM → Checkpoint → ตั้งชื่อ: "Clean CentOS Stream 10"
```

![สร้าง Snapshot](images/lab00-41.png)

![Snapshot เสร็จแล้ว](images/lab00-42.png)

หาก DevStack ติดตั้งไม่สำเร็จ สามารถ Restore กลับมาจุดนี้ได้

---

## ตารางสรุปการตั้งค่าเครือข่าย

| Adapter | Virtual Switch | Interface | IP Address | หมายเหตุ |
|---------|---------------|-----------|------------|----------|
| Adapter 1 | DefaultSwitch | enp0s3 | DHCP | เชื่อมต่ออินเทอร์เน็ต |
| Adapter 2 | Internal-Mgmt | enp0s4 | 10.20.30.11/24 | Management Network / Horizon |
| Adapter 3 | Internal-BrEx | enp0s5 | 192.168.56.11/24 | Neutron br-ex bridge |

---

## ภาพประกอบเพิ่มเติม

![เพิ่มเติม 1](images/lab00-43.png)

![เพิ่มเติม 2](images/lab00-44.png)

![เพิ่มเติม 3](images/lab00-45.png)

![เพิ่มเติม 4](images/lab00-46.png)

![เพิ่มเติม 5](images/lab00-47.png)

![เพิ่มเติม 6](images/lab00-48.png)

![เพิ่มเติม 7](images/lab00-49.png)

![เพิ่มเติม 8](images/lab00-50.png)

![เพิ่มเติม 9](images/lab00-51.png)

---

## เอกสารอ้างอิง

- [CentOS Stream 10 Installation Guide](https://docs.centos.org/en-US/stream/install/)
- [Microsoft Hyper-V Documentation](https://learn.microsoft.com/en-us/virtualization/hyper-v-on-windows/)
- [DevStack on CentOS Stream](https://docs.openstack.org/devstack/latest/)

---

## ตรวจสอบก่อนไป Lab ถัดไป

ก่อนดำเนินการ [Lab 1: ติดตั้ง DevStack](lab01-devstack.md) ให้ตรวจสอบรายการนี้:

- [x] CentOS Stream 10 ติดตั้งเสร็จสมบูรณ์
- [x] ทั้ง 3 network interfaces ตอบสนอง (ใช้ `ip addr` ตรวจสอบ)
- [x] สามารถ ping ออกอินเทอร์เน็ตได้
- [x] SSH เข้าเครื่องได้จาก Host (ผ่าน IP ของ enp0s3 หรือ enp0s4)
- [x] สร้าง Snapshot "Clean CentOS Stream 10" แล้ว
- [x] ผู้ใช้ `stack` มีสิทธิ์ sudo โดยไม่ต้องใส่รหัสผ่าน
- [x] SELinux อยู่ในโหมด permissive
- [x] Firewall ปิด (หรือเปิดพอร์ตที่จำเป็น)
