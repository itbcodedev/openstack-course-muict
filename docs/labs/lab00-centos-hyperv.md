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

![](./images/lab00-1.png)

***Figure 1 กดปุม ทางขวา มือ Virtual Switch Manager สร้าง internal switch ทำหน้าการเชื่อมต่อ ใน layer 2 , และไม่มี DHCP Server เพื่อให้มีการร้องขอ ip ต้องกำหนดเอง***

![](./images/lab00-2.png)
***Figure 2 สร้าง Internal Virtual Switch จะสร้าง 2 Switch คือ Internal-Mgmt-Net และ Internal-BrEx-Net***

### 1. Default Switch (เชื่อมต่ออินเทอร์เน็ต)

Hyper-V สร้าง Default Switch มาให้อัตโนมัติ หากไม่มี ให้สร้าง External Switch ที่เชื่อมต่อกับการ์ดเครือข่ายหลัก:

```
Virtual Switch Manager → New virtual network switch → External → Create Virtual Switch
ชื่อ: DefaultSwitch (หรือ External)
Connection type: External network → [เลือกการ์ดเครือข่ายหลัก]
☑ Allow management operating system to share this network adapter
```

### 2. Internal Management Network

![](./images/lab00-3.png)

***Figure 3 สร้าง  Internal-Mgmt-Net ให้ตั้งชื่อ อย่างเดียว ส่วน Note จะมีหรือไม่มีก็ได้ ไม่มีผล และให้กด Apply เพื่อเป็นการยืนยัน***


```
Virtual Switch Manager → New virtual network switch → Internal → Create Virtual Switch
ชื่อ: Internal-Mgmt-Net
Connection type: Internal network
```

- ใช้สำหรับเข้าถึงหน้า Web OpenStack (Horizon Dashboard) และ API ของ OpenStack
- IP ภายในเครือข่าย เช่น 10.20.30.0/24

### 3. Internal External Network (สำหรับ br-ex ของ Neutron Router)

![](./images/lab00-4.png)

***Figure 4 สร้าง  Internal-BrEx-Net***

```
Virtual Switch Manager → New virtual network switch → Internal → Create Virtual Switch
ชื่อ: Internal-BrEx-Net
Connection type: Internal network
```

- ใช้เป็น physical interface สำหรับ Neutron br-ex bridge
- จะกำหนดค่า IP ให้กับอินเทอร์เฟซนี้เมื่อติดตั้ง OpenStack ด้วย DevStack
- IP ภายในเครือข่าย เช่น 192.168.56.0/24

> **ตรวจสอบ Virtual Switch:** หลังจากสร้างเสร็จ ให้ตรวจสอบว่ามีทั้งหมด 3 switches:
> - DefaultSwitch (หรือ External)
> - Internal-Mgmt
> - Internal-BrEx

---

## สร้าง Virtual Machine

### ขั้นตอนที่ 1: New Virtual Machine Wizard

![](./images/lab00-5.png)

```
Hyper-V Manager → Action → New → Virtual Machine
```
![](./images/lab00-6.png)

![](./images/lab00-7.png)

![](./images/lab00-8.png)

![](./images/lab00-9.png)

![](./images/lab00-10.png)

![](./images/lab00-11.png)

![](./images/lab00-12.png)

| ขั้นตอน | ค่าที่ตั้ง |
|---------|-----------|
| Name | DevStackCentos10Stream |
| location | D:\Hyper-V\ |
| Generation | Generation 1 |
| Memory | 8192 MB (8 GB) — ปิด Dynamic memory (คลิกเครื่องหมายถูกออก) |
| Network | เลือก DefaultSwitch ก่อน (จะเพิ่มอีก 2 adapters ในขั้นตอนหลัง) |
| Hard Disk | Create a virtual hard disk: 60 GB (VHDX) |
| Installation Options | Attach ISO: CentOS-Stream-10-latest-x86_64-dvd1.iso |

> **สำคัญ: ปิด Dynamic Memory —** OpenStack ต้องการหน่วยความจำคงที่ ให้ปิด Dynamic memory แล้วตั้งค่า Startup RAM = 8192 MB

### ขั้นตอนที่ 2: เพิ่ม Network Adapter

หลังจากสร้าง VM เสร็จ ให้เปิด **Settings** ของ VM และเพิ่ม Network Adapter อีก 2 ตัว:

![](./images/lab00-13.png)

```
Settings → Add Hardware → Network Adapter → Add
```

- **Network Adapter 1:** DefaultSwitch (มีอยู่แล้ว)
- **Network Adapter 2:** Internal-Mgmt
- **Network Adapter 3:** Internal-BrEx

![](./images/lab00-14.png)

![](./images/lab00-15.png)

![](./images/lab00-16.png)


รวมทั้งหมด 3 Network Adapters
![](./images/lab00-17.png)

### ขั้นตอนที่ 3: ปรับแต่งเพิ่มเติม

![](./images/lab00-18.png)

```
Settings:
├── Security
│   └── Secure Boot: ☐ Uncheck (ปิด Secure Boot — จำเป็นสำหรับ Linux บน Gen 2)
├── Processor
│   └── Number of virtual processors: 8 (หรือ 4 หากมีไม่พอ)
├── Integration Services
│   └── ☑ Guest services (ช่วยให้ Copy-Paste ระหว่าง Host กับ VM ได้)
└── Checkpoint File Location
    └── กำหนดพื้นที่จัดเก็บ Snapshot
```

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

> **จด IP ไว้ใช้ในภายหลัง:**
> - enp0s3 (Default): DHCP - รับ IP อัตโนมัติ
> - enp0s4 (Mgmt): 10.20.30.11/24
> - enp0s5 (BrEx): 192.168.56.11/24 — ต้องปิด or ลบ IP นี้ออกหลังติดตั้ง DevStack

#### ✅ Time & Date

```
Time & Date → Region: Asia → City: Bangkok → Network Time: ON → Done
```

#### ✅ Software Selection

```
Software Selection → Server with GUI → Done
```

แนะนำเลือก **Server with GUI** เพื่อความสะดวกในการเรียนรู้ หรือเลือก **Minimal Install** หากต้องการเบา

#### ✅ Root Password

```
Root Password: ตั้งรหัสผ่านสำหรับ root
```

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

> **แนะนำใช้ชื่อผู้ใช้ stack —** DevStack ตั้งค่าให้ติดตั้งภายใต้ผู้ใช้ /opt/stack โดยอัตโนมัติ การใช้ชื่อ **stack** จะทำให้สะดวกขึ้น

### ขั้นตอนที่ 3: เริ่มติดตั้ง

```
→ Begin Installation
```

รอประมาณ 10-20 นาทีให้การติดตั้งเสร็จสิ้น จากนั้น:

```
→ Reboot System
```

!!hint ตรวจสอบ ISO หลัง Reboot
    หลังจาก Reboot ให้ปิด VM → Settings → DVD Drive → ลบ ISO ออก → เปิด VM ใหม่

---

## ตั้งค่าหลังติดตั้ง (Post-Installation)

### ขั้นตอนที่ 1: เข้าสู่ระบบ

```
Username: stack
Password: [รหัสที่ตั้งไว้]
```

### ขั้นตอนที่ 2: ตรวจสอบ Network Interfaces

เปิด Terminal แล้วรันคำสั่ง:

```bash
ip addr show
```

ควรเห็น 3 interfaces:
- `enp0s3`: มี IP จาก DHCP (เชื่อมต่ออินเทอร์เน็ตได้)
- `enp0s4`: 10.20.30.11 — Management Network
- `enp0s5`: 192.168.56.11 — External Network (สำหรับ br-ex)

### ขั้นตอนที่ 3: อัปเดตระบบ

```bash
sudo dnf update -y
sudo dnf install -y epel-release git vim wget curl net-tools bash-completion
```

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

### ขั้นตอนที่ 5: ปิด SELinux (สำหรับ Lab)

```bash
sudo setenforce 0
sudo sed -i 's/SELINUX=enforcing/SELINUX=permissive/g' /etc/selinux/config
```

> **ห้ามปิด SELinux บน Production —** ในสภาพแวดล้อมจริงควรปรับแต่ง SELinux policy แทนการปิด

### ขั้นตอนที่ 6: ตั้งค่า sudo สำหรับ stack user

```bash
sudo visudo
```

เพิ่มบรรทัดนี้ลงไป:
```
stack ALL=(ALL) NOPASSWD: ALL
```

### ขั้นตอนที่ 7: สร้าง Snapshot

ก่อนติดตั้ง DevStack ให้สร้าง Checkpoint (Snapshot) ไว้:

```
Hyper-V Manager → เลือก VM → Checkpoint → ตั้งชื่อ: "Clean CentOS Stream 10"
```

หาก DevStack ติดตั้งไม่สำเร็จ สามารถ Restore กลับมาจุดนี้ได้

---

## ตารางสรุปการตั้งค่าเครือข่าย

| Adapter | Virtual Switch | Interface | IP Address | หมายเหตุ |
|---------|---------------|-----------|------------|----------|
| Adapter 1 | DefaultSwitch | enp0s3 | DHCP | เชื่อมต่ออินเทอร์เน็ต |
| Adapter 2 | Internal-Mgmt | enp0s4 | 10.20.30.11/24 | Management Network / Horizon |
| Adapter 3 | Internal-BrEx | enp0s5 | 192.168.56.11/24 | Neutron br-ex bridge |

---

## ภาพประกอบ (รอการฝัง)

> **อาจารย์จะ Crop หน้าจอจากขั้นตอนต่าง ๆ มาใส่ในส่วนนี้**

### ภาพที่ 1: หน้าจอสร้าง Virtual Switch
```
[สำหรับใส่ Screenshot: Virtual Switch Manager แสดงทั้ง 3 switches]
```

### ภาพที่ 2: หน้าจอสร้าง VM Generation 2
```
[สำหรับใส่ Screenshot: New Virtual Machine Wizard - Generation 2]
```

### ภาพที่ 3: หน้าจอ Settings แสดง 3 Network Adapters
```
[สำหรับใส่ Screenshot: VM Settings - 3 Network Adapters]
```

### ภาพที่ 4: หน้าจอติดตั้ง CentOS Stream 10
```
[สำหรับใส่ Screenshot: Install CentOS Stream 10]
```

### ภาพที่ 5: หน้าจอ Network & Host Name
```
[สำหรับใส่ Screenshot: Network Configuration - 3 interfaces configured]
```

### ภาพที่ 6: หน้าจอ Installation Summary พร้อมตั้งค่าครบถ้วน
```
[สำหรับใส่ Screenshot: INSTALLATION SUMMARY - all items configured]
```

### ภาพที่ 7: หน้าจอหลังติดตั้งเสร็จ — เข้าใช้งาน CentOS Stream 10
```
[สำหรับใส่ Screenshot: CentOS Stream 10 Desktop with Terminal showing ip addr]
```

---

## เอกสารอ้างอิง

- [CentOS Stream 10 Installation Guide](https://docs.centos.org/en-US/stream/install/)
- [Microsoft Hyper-V Documentation](https://learn.microsoft.com/en-us/virtualization/hyper-v-on-windows/)
- [DevStack on CentOS Stream](https://docs.openstack.org/devstack/latest/)

---

## ตรวจสอบก่อนไป Lab ถัดไป

ก่อนดำเนินการ [Lab 1: ติดตั้ง DevStack](lab01-devstack.md) ให้ตรวจสอบรายการนี้:

- [ ] CentOS Stream 10 ติดตั้งเสร็จสมบูรณ์
- [ ] ทั้ง 3 network interfaces ตอบสนอง (ใช้ `ip addr` ตรวจสอบ)
- [ ] สามารถ ping ออกอินเทอร์เน็ตได้
- [ ] SSH เข้าเครื่องได้จาก Host (ผ่าน IP ของ enp0s3 หรือ enp0s4)
- [ ] สร้าง Snapshot "Clean CentOS Stream 10" แล้ว
- [ ] ผู้ใช้ `stack` มีสิทธิ์ sudo โดยไม่ต้องใส่รหัสผ่าน
- [ ] SELinux อยู่ในโหมด permissive
- [ ] Firewall ปิด (หรือเปิดพอร์ตที่จำเป็น)
