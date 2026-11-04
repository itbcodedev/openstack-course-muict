# การบรรยายที่ 7: Cinder — บริการ Block Storage

## วัตถุประสงค์การเรียนรู้

- เข้าใจบทบาทของ Cinder ในการจัดสรร Block Storage
- รู้จัก Backend Drivers: LVM, NFS, Ceph, iSCSI
- สร้างและจัดการ Volumes, Snapshots และ Backups
- ผูก Volume เข้ากับ VM (Attach/Detach)

---

## Cinder คืออะไร?

**Cinder** เป็นบริการ Block Storage ของ OpenStack ให้บริการ:

- **Persistent Block Devices** — ดิสก์ที่คงทนของ VM (คงอยู่แม้ลบ VM)
- **Volume Lifecycle** — Create, Attach, Detach, Extend, Snapshot, Backup, Delete
- **Multiple Backends** — LVM, NFS, Ceph RBD, iSCSI, FC
- **Storage QoS** — กำหนดคุณภาพบริการของ Storage

---

## สถาปัตยกรรม Cinder

```
┌─────────────────────────────────────────┐
│         cinder-api                      │
│   (รับ REST API requests)               │
└──────────────────┬──────────────────────┘
                   │
┌──────────────────▼──────────────────────┐
│         cinder-scheduler                │
│   (เลือก Backend ที่เหมาะสม)            │
└──────────────────┬──────────────────────┘
                   │ ผ่าน Message Queue
┌──────────────────▼──────────────────────┐
│         cinder-volume                   │
│   (จัดการ Volume บน Backend Storage)    │
│   - LVM, Ceph, NFS, iSCSI, FC           │
├─────────────────────────────────────────┤
│         cinder-backup                   │
│   (สำรอง Volume ไปยัง Object Store)      │
└─────────────────────────────────────────┘
```

### ส่วนประกอบ

| ส่วนประกอบ | หน้าที่ |
|-----------|------|
| **cinder-api** | รับและตรวจสอบ API requests |
| **cinder-scheduler** | เลือก Backend Host ที่เหมาะสม |
| **cinder-volume** | จัดการ Volume บน Storage Backend โดยตรง |
| **cinder-backup** | สำรอง Volume ไปยัง Swift หรือ Object Store |

---

## Volume Types

**Volume Type** คือ Template ที่ผูกกับ Backend และคุณสมบัติ:

```bash
# แสดงรายการ Volume Types
openstack volume type list

# สร้าง Volume Type
openstack volume type create --description "SSD Fast Tier" ssd-fast

# ตั้งค่า Extra Specs (QoS)
cinder type-key ssd-fast set volume_backend_name=LVM-SSD
```

| คุณสมบัติ | คำอธิบาย |
|----------|---------|
| `volume_backend_name` | ชื่อ Backend ที่จะใช้ |
| `provisioned_iops` | IOPS ที่รับประกัน |
| `multiattach` | รองรับการ Attach หลาย VM |

---

## การจัดการ Volumes

```bash
# สร้าง Volume
openstack volume create --size 10 --type ssd-fast my-volume

# แสดงรายการ Volumes
openstack volume list

# ดูรายละเอียด Volume
openstack volume show my-volume

# Extend (ขยายขนาด)
openstack volume set --size 20 my-volume

# ลบ Volume
openstack volume delete my-volume
```

---

## Attach / Detach Volume

```
┌─────────────────────────────────────────┐
│         Compute Node                     │
│  ┌─────────┐      ┌────────────────┐   │
│  │   VM    │ ←──→ │ /dev/vdb (VirtIO)│  │
│  │ (Nova)  │      │ Cinder Volume   │   │
│  └─────────┘      └────────────────┘   │
└─────────────────────────────────────────┘
```

```bash
# ผูก Volume กับ VM
openstack server volume attach my-instance my-volume

# ภายใน VM — ตรวจสอบดิสก์ใหม่
lsblk
sudo fdisk -l

# สร้าง Filesystem บน Volume
sudo mkfs.ext4 /dev/vdb
sudo mkdir /mnt/data
sudo mount /dev/vdb /mnt/data

# ถอด Volume (ต้อง Unmount ก่อน)
sudo umount /mnt/data
openstack server volume detach my-instance my-volume
```

---

## Snapshots

Snapshot บันทึกสถานะ Volume ณ เวลาหนึ่ง:

```bash
# สร้าง Snapshot
openstack snapshot create --volume my-volume my-snapshot

# แสดงรายการ Snapshots
openstack snapshot list

# สร้าง Volume ใหม่จาก Snapshot
openstack volume create --snapshot my-snapshot --size 10 restored-volume

# ลบ Snapshot
openstack snapshot delete my-snapshot
```

!!! warning "อย่าสับสน"
    **Volume Snapshot** (Cinder) ≠ **VM Snapshot** (Nova Image)  
    Volume Snapshot = สำรองดิสก์อย่างเดียว  
    VM Snapshot = สำรองดิสก์ + สถานะ VM (RAM, CPU)

---

## Backups

Backup คือการสำรอง Volume ไปยัง Object Store (Swift):

```bash
# สร้าง Backup
openstack volume backup create --volume my-volume my-backup

# แสดงรายการ Backups
openstack volume backup list

# Restore จาก Backup
openstack volume backup restore my-backup new-volume
```

| คุณสมบัติ | Snapshot | Backup |
|----------|---------|--------|
| ที่เก็บ | บน Backend เดียวกัน | Object Store (offsite) |
| ใช้เวลา | เร็ว (CoW) | ช้ากว่า |
| ความปลอดภัย | ถ้า Backend พัง = หาย | กระจายความเสี่ยง |

---

## Boot from Volume

สร้าง VM โดยใช้ Volume เป็นดิสก์เริ่มต้นแทน Image:

```bash
# สร้าง Bootable Volume จาก Image
openstack volume create --image cirros --size 5 --bootable vol-cirros

# Boot VM จาก Volume
openstack server create \
  --flavor m1.tiny \
  --volume vol-cirros \
  --network private-net \
  --key-name my-key \
  vm-from-volume
```

!!! tip "ข้อดีของ Boot from Volume"
    - VM ย้ายได้ระหว่าง Host (Live Migration)
    - ดิสก์อยู่บน Shared Storage (Ceph, NFS)
    - ง่ายต่อการสำรองและ Restore

---

## สไลด์การบรรยาย

📥 [ดาวน์โหลดสไลด์การบรรยายที่ 7 (PDF)](../assets/slides/OpenStack_Cinder_Architecture.pdf){ .md-button }

## แหล่งข้อมูลเพิ่มเติม

- [Cinder Documentation](https://docs.openstack.org/cinder/latest/)
- [Cinder Admin Guide](https://docs.openstack.org/cinder/latest/admin/)
