# การบรรยายที่ 4: Nova — บริการ Compute

## วัตถุประสงค์การเรียนรู้

- เข้าใจบทบาทและสถาปัตยกรรมของ Nova
- รู้จักส่วนประกอบหลัก: API, Scheduler, Compute, Conductor
- อธิบายกระบวนการสร้าง VM ตั้งแต่ Request จนถึง Boot
- จัดการ Flavors, Keypairs, Security Groups และ Quotas

---

## Nova คืออะไร?

**Nova** เป็นบริการ Compute ของ OpenStack รับผิดชอบ:

- จัดเตรียมและจัดการ Virtual Machines (Instances)
- รองรับหลาย Hypervisor: **KVM**, Xen, VMware, Hyper-V, LXC
- ทำงานแบบ Stateless — สถานะอยู่ใน Database
- สื่อสารผ่าน Message Queue (RabbitMQ)

---

## สถาปัตยกรรม Nova

```
┌─────────────────────────────────────────┐
│           nova-api                      │
│     (รับ REST API / Validate Request)   │
└──────────────────┬──────────────────────┘
                   │
┌──────────────────▼──────────────────────┐
│       nova-scheduler                    │
│   (Filter → Weigh → เลือก Host)        │
└──────────────────┬──────────────────────┘
                   │ ผ่าน RabbitMQ
┌──────────────────▼──────────────────────┐
│       nova-compute บน Host ที่เลือก      │
│   (Libvirt + KVM สร้าง VM)              │
└─────────────────────────────────────────┘
```

### ส่วนประกอบหลัก

| ส่วนประกอบ | หน้าที่ |
|-----------|------|
| **nova-api** | รับ API requests, ตรวจสอบสิทธิ์, ส่งต่อไปยัง Message Queue |
| **nova-scheduler** | ตัดสินใจเลือก Compute Host ที่เหมาะสม |
| **nova-compute** | จัดการ VMs บน Hypervisor โดยตรง |
| **nova-conductor** | ตัวกลางเข้าถึง Database ลดการเชื่อมต่อโดยตรงจาก Compute |
| **nova-console** | Proxy VNC / SPICE / Serial Console |

---

## กระบวนการสร้าง VM (Boot Instance)

```
User → Horizon / CLI ส่งคำขอ boot
  ↓
nova-api → Keystone (ตรวจสอบ Token)
  ↓
nova-api ส่งข้อความไป Message Queue
  ↓
nova-scheduler รับข้อความ
  ↓
Filter Scheduler → กรอง Host ที่ไม่เหมาะสม
  ↓
Weight Scheduler → ให้คะแนน Host (RAM, CPU ว่าง)
  ↓
เลือก Host → ส่งข้อความไปยัง nova-compute บน Host นั้น
  ↓
nova-compute:
  1. ขอ Image จาก Glance → ดาวน์โหลดไปยัง /var/lib/nova/instances
  2. ขอ Network จาก Neutron → สร้าง vNIC + Port
  3. ขอ Volume จาก Cinder (ถ้ามี) → Attach
  4. สั่ง libvirt สร้าง Domain XML → สั่ง boot VM
  ↓
VM เริ่มทำงาน → Cloud-init ตั้งค่าเริ่มต้น
```

---

## Flavors

**Flavor** คือ Template ที่กำหนด Spec ของ VM:

| พารามิเตอร์ | คำอธิบาย |
|-----------|------|
| **VCPUs** | จำนวน vCPU |
| **RAM (MB)** | ขนาดหน่วยความจำ |
| **Root Disk (GB)** | ขนาดดิสก์เริ่มต้น |
| **Ephemeral Disk** | ดิสก์ชั่วคราว (หายเมื่อลบ VM) |
| **Swap** | ขนาด Swap |
| **RX/TX Factor** | อัตราคูณแบนด์วิดธ์ |

```bash
# แสดงรายการ Flavors
openstack flavor list

# สร้าง Flavor ใหม่
openstack flavor create --ram 2048 --disk 20 --vcpus 2 m1.small-custom
```

---

## Keypairs

Keypair ใช้สำหรับเข้าถึง VM ผ่าน SSH โดยไม่ต้องใช้ Password:

```bash
# สร้าง Keypair
openstack keypair create my-key > my-key.pem
chmod 600 my-key.pem

# นำ Public Key ที่มีอยู่แล้วเข้าสู่ระบบ
openstack keypair create --public-key ~/.ssh/id_rsa.pub existing-key

# แสดงรายการ Keypairs
openstack keypair list
```

!!! warning "สำคัญ"
    เก็บ Private Key ให้ปลอดภัย — OpenStack ไม่เก็บสำเนา Private Key

---

## Quotas

Quota จำกัดทรัพยากรที่ Project สามารถใช้ได้:

| Quota | ค่าเริ่มต้น | คำอธิบาย |
|-------|-----------|------|
| instances | 10 | จำนวน VM สูงสุด |
| cores | 20 | จำนวน vCPU สูงสุด |
| ram | 51200 (MB) | RAM รวมสูงสุด |
| floating_ips | 10 | จำนวน Floating IP |
| fixed_ips | -1 (unlimited) | จำนวน Fixed IP |

```bash
# แสดง Quotas ของ Project
openstack quota show <project-id>

# แก้ไข Quota
openstack quota set --instances 20 --cores 40 <project-id>
```

---

## การจัดการ VMs

```bash
# สร้าง VM
openstack server create \
  --image cirros \
  --flavor m1.tiny \
  --network private-net \
  --key-name my-key \
  my-instance

# แสดงรายการ VMs
openstack server list

# ดูรายละเอียด VM
openstack server show my-instance

# เริ่ม / หยุด / รีสตาร์ท / ลบ
openstack server start my-instance
openstack server stop my-instance
openstack server reboot my-instance
openstack server delete my-instance

# Resize (เปลี่ยน Flavor)
openstack server resize --flavor m1.small my-instance
openstack server resize confirm my-instance   # ยืนยัน
openstack server resize revert my-instance    # ย้อนกลับ

# สร้าง Snapshot
openstack server image create --name my-snapshot my-instance
```

---

## Live Migration

ย้าย VM จาก Host หนึ่งไปยังอีก Host โดยไม่ต้อง Shutdown:

| ประเภท | เงื่อนไข | ข้อจำกัด |
|--------|---------|---------|
| **Live Migration** | Shared Storage + ชนิด CPU ใกล้เคียงกัน | ไม่สามารถ Live Migrate ถ้ามี CD-ROM หรือ PCI Passthrough |
| **Cold Migration** | ปิด VM ก่อนย้าย | ใช้งานได้ทุกกรณี |

```bash
# Live Migrate
openstack server migrate --live <destination-host> my-instance

# Cold Migrate
openstack server migrate my-instance
```

---

## สไลด์การบรรยาย

📥 [ดาวน์โหลดสไลด์การบรรยายที่ 4 (PDF)](../assets/slides/lecture04.pdf){ .md-button }

## แหล่งข้อมูลเพิ่มเติม

- [Nova Documentation](https://docs.openstack.org/nova/latest/)
- [Nova Admin Guide](https://docs.openstack.org/nova/latest/admin/)
