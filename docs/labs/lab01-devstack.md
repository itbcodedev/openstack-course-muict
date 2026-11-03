# การทดลองที่ 1: ติดตั้ง DevStack บน CentOS Stream 10

## วัตถุประสงค์

ติดตั้งสภาพแวดล้อม OpenStack แบบ All-in-One บน CentOS Stream 10 โดยใช้ DevStack บน Virtual Machine ที่สร้างจาก [Lab 0: ติดตั้ง CentOS Stream 10 บน Hyper-V](lab00-centos-hyperv.md)

---

## ข้อกำหนด

| รายการ | ข้อกำหนด |
|--------|-----------|
| ระบบปฏิบัติการ | CentOS Stream 10 (ติดตั้งจาก Lab 0) |
| หน่วยความจำ (RAM) | 8 GB (8192 MB) |
| ดิสก์ | 60 GB |
| ผู้ใช้ | `stack` (non-root พร้อม sudo NOPASSWD) |
| เครือข่าย | 3 adapters ตามที่ตั้งค่าใน Lab 0 |

---

## Step 1: เตรียมระบบ

### สลับไปยังผู้ใช้ stack

```bash
sudo -u stack -i
```

หรือหากยังไม่ได้สร้างผู้ใช้ stack:

```bash
sudo useradd -s /bin/bash -d /opt/stack -m stack
echo "stack ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/stack
sudo -u stack -i
```

### ตรวจสอบ IP addresses

```bash
ip addr show
```

ควรเห็น 3 interfaces:
- `enp0s3`: IP จาก DHCP (อินเทอร์เน็ต)
- `enp0s4`: 10.20.30.11 (Management)
- `enp0s5`: 192.168.56.11 (External / สำหรับ br-ex)

---

## Step 2: Clone DevStack

```bash
cd /opt/stack
git clone https://opendev.org/openstack/devstack
cd devstack
```

---

## Step 3: กำหนดค่า DevStack

สร้างไฟล์ `local.conf` โดยใช้ค่าที่เหมาะสมกับเครือข่าย 3 ตัวบน Hyper-V:

```bash
cat > local.conf << 'EOF'
[[local|localrc]]
ADMIN_PASSWORD=Muict@2024
DATABASE_PASSWORD=$ADMIN_PASSWORD
RABBIT_PASSWORD=$ADMIN_PASSWORD
SERVICE_PASSWORD=$ADMIN_PASSWORD
SERVICE_TOKEN=a682f596-76f3-11ef-9f11-0242ac120003

# IP ของ Management Interface (enp0s4)
HOST_IP=10.20.30.11

# บริการที่ต้องการเปิดใช้งาน
enable_service rabbit mysql key
enable_service n-api n-cpu n-cond n-sch n-novnc n-cauth
enable_service g-api g-reg
enable_service cinder c-sch c-api c-vol
enable_service neutron q-svc q-agt q-dhcp q-l3 q-meta

# ===== Neutron Networking สำหรับ Hyper-V 3 Adapters =====
# External Network สำหรับ Floating IPs
FLOATING_RANGE=192.168.56.0/24

# Private Network สำหรับ VM ภายใน
FIXED_RANGE=10.0.0.0/24
FIXED_NETWORK_SIZE=256

# Physical interface สำหรับ br-ex bridge (enp0s5)
FLAT_INTERFACE=enp0s5
PUBLIC_INTERFACE=enp0s5

# External Bridge
PUBLIC_BRIDGE=br-ex

# ปิดบริการที่ไม่จำเป็นสำหรับ Lab
disable_service tempest

# การบันทึก Log
LOGFILE=$DEST/logs/stack.sh.log
LOGDAYS=2

# ไม่ต้องใช้ Floating IP อัตโนมัติ
# (จะสร้างและกำหนดเองใน Lab ถัดไป)
EOF
```

!!! tip "สำคัญ: ปรับ HOST_IP"
    ปรับ `HOST_IP` ให้ตรงกับ IP ของ Management Interface (enp0s4) บน VM ของคุณ
    ใช้คำสั่ง `ip addr show enp0s4` เพื่อตรวจสอบ

!!! tip "รหัสผ่าน"
    เปลี่ยน `ADMIN_PASSWORD` ให้ปลอดภัย ในตัวอย่างใช้ `Muict@2024` ค่าอื่น ๆ จะใช้รหัสเดียวกันเพื่อความสะดวกใน Lab

---

## Step 4: รันคำสั่งติดตั้ง DevStack

```bash
cd /opt/stack/devstack
./stack.sh
```

การติดตั้งจะใช้เวลา **30–90 นาที** ขึ้นอยู่กับความเร็วอินเทอร์เน็ตและฮาร์ดแวร์

หากสำเร็จ จะแสดงข้อความคล้ายกับนี้:

```
=========================
DevStack Component Timing
=========================
Total runtime: 2847 seconds
...
This is your host IP address: 10.20.30.11
This is your host IPv6 address: ::1
Horizon is now available at http://10.20.30.11/dashboard
Keystone is serving at http://10.20.30.11/identity/
The default users are: admin and demo
The password: Muict@2024
```

---

## Step 5: ตรวจสอบการติดตั้ง

### โหลด Credentials

```bash
cd /opt/stack/devstack
source openrc admin
```

### ตรวจสอบบริการต่าง ๆ

```bash
# รายการบริการ Compute
openstack compute service list

# รายการ Network Agent
openstack network agent list

# รายการ VM (ควรว่าง)
openstack server list

# รายการรูปแบบ (Flavors)
openstack flavor list

# รายการรูป VM (Images)
openstack image list
```

### ตรวจสอบ Horizon Dashboard

เปิดเว็บเบราว์เซอร์บนเครื่อง Host (Windows) และเข้าไปที่:

```
http://10.20.30.11/dashboard
```

| ช่อง | ค่า |
|------|-----|
| Domain | `default` |
| User Name | `admin` |
| Password | `Muict@2024` (หรือ ADMIN_PASSWORD ที่ตั้งไว้) |

---

## Step 6: สร้าง VM ตัวแรกผ่าน CLI

```bash
# โหลด credentials ของ demo user
source /opt/stack/devstack/openrc demo demo

# สร้าง Keypair
ssh-keygen -t rsa -f ~/.ssh/devstack_key -N ""
openstack keypair create --public-key ~/.ssh/devstack_key.pub devstack_key

# เปิด Security Group สำหรับ SSH
openstack security group rule create --protocol tcp --dst-port 22 \
  --remote-ip 0.0.0.0/0 default

#  запустить VM
openstack server create \
  --image cirros-0.6.2-x86_64-disk \
  --flavor m1.tiny \
  --network private \
  --key-name devstack_key \
  --security-group default \
  --wait \
  vm-01

# ตรวจสอบสถานะ
openstack server list
```

---

## ปัญหาที่พบบ่อย

| ปัญหา | วิธีแก้ไข |
|-------|-----------|
| `stack.sh` ล้มเหลว มีข้อผิดพลาดเกี่ยวกับ apt | CentOS ใช้ `dnf` ไม่ใช่ `apt` — DevStack รองรับอัตโนมัติ แต่ตรวจสอบว่ามีอินเทอร์เน็ต |
| หน่วยความจำไม่พอ OOM | เพิ่ม RAM VM เป็น 12-16 GB หรือเพิ่ม Swap |
| เครือข่ายไม่สามารถเชื่อมต่อได้ | ตรวจสอบ `HOST_IP` และ `FLAT_INTERFACE` ใน local.conf |
| ติดตั้งไม่สำเร็จ | รัน `./unstack.sh && ./stack.sh` อีกครั้ง |
| Neutron agent รายงาน XXX | ตรวจสอบว่า enp0s5 (FLAT_INTERFACE) อยู่ใน state UP |
| ไม่สามารถเข้า Horizon ได้ | ตรวจสอบ firewall — `sudo systemctl status firewalld` |

---

## การล้างข้อมูลและเริ่มใหม่

```bash
cd /opt/stack/devstack
./unstack.sh      # หยุดบริการทั้งหมด
./clean.sh        # ล้างข้อมูลทั้งหมด (ลบทุกอย่าง)
```

หากต้องการเริ่มต้นใหม่ทั้งหมด แนะนำให้ **Restore Snapshot** จาก Hyper-V Manager แทน:

```
Hyper-V Manager → คลิกขวาที่ VM → Restore → เลือก "Clean CentOS Stream 10"
```

---

## สรุปการตั้งค่าเครือข่าย

| อินเทอร์เฟซ | Virtual Switch | IP | บทบาทใน OpenStack |
|-----------|---------------|-----|-------------------|
| enp0s3 | DefaultSwitch | DHCP | อินเทอร์เน็ต/ดาวน์โหลด |
| enp0s4 | Internal-Mgmt | 10.20.30.11 | Horizon/API, HOST_IP |
| enp0s5 | Internal-BrEx | 192.168.56.11 | br-ex, Floating IPs |

---

## การบ้าน

📋 **รายงานการทดลอง:**

1. Screenshot หน้า Horizon Dashboard หลังจากเข้าสู่ระบบสำเร็จ
2. Screenshot รายการ `openstack compute service list`
3. Screenshot รายการ `openstack network agent list`
4. Screenshot `openstack server list` แสดง vm-01 กำลังทำงาน
5. ตอบคำถาม: `HOST_IP` ใน `local.conf` มีหน้าที่อะไร?
6. ตอบคำถาม: ทำไมต้องมี 2 อินเทอร์เฟซแยกกัน (Mgmt vs External)?

ส่งผ่าน: [Google Classroom / LMS]
