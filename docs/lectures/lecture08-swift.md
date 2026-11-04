# การบรรยายที่ 8: Swift — บริการ Object Storage

## วัตถุประสงค์การเรียนรู้

- เข้าใจความแตกต่างระหว่าง Block, File และ Object Storage
- รู้จักสถาปัตยกรรม Ring และ Data Distribution ของ Swift
- อธิบายการ Replicate และ Consistency ใน Object Store
- ใช้งาน Swift CLI และ API เบื้องต้น

---

## Swift คืออะไร?

**Swift** เป็นบริการ Object Storage ของ OpenStack:

- **Scalable** — ขยายได้โดยเพิ่ม Node ใหม่ (Scale-out)
- **Highly Available** — ข้อมูล Replicate อัตโนมัติ
- **Durable** — รองรับ Hardware Failure โดยไม่สูญหาย
- **No Single Point of Failure** — ทุกส่วนเป็น Redundant
- **REST API** — HTTP-based (S3-compatible)

!!! info "Object Storage ≠ Block Storage"
    | Object Storage | Block Storage |
    |----------------|---------------|
    | เก็บ Object (ไฟล์ + Metadata) | เก็บ Block (raw disk) |
    | Flat namespace (ไม่มี folder ซ้อน) | Filesystem hierarchy |
    | HTTP API | iSCSI/FC/NFS |
    | ขยายได้ไม่จำกัด | จำกัดตามขนาดดิสก์ |
    | Best for: Backups, Archives, Media | Best for: Databases, VMs |

---

## สถาปัตยกรรม Swift

```
┌─────────────────────────────────────────┐
│         Proxy Node (Load Balancer)      │
│         swift-proxy-server               │
│         nginx / haproxy                  │
└──────────────────┬──────────────────────┘
                   │
┌──────────────────▼──────────────────────┐
│         Storage Node 1                   │
│  ┌────────┐ ┌────────┐ ┌────────┐       │
│  │ Account │ │Container│ │ Object │       │
│  │ Server  │ │ Server  │ │ Server │       │
│  └────────┘ └────────┘ └────────┘       │
│  ┌────────┐ ┌────────┐                  │
│  │ Account │ │Container│                  │
│  │  Ring   │ │  Ring   │                  │
│  └────────┘ └────────┘                  │
│  ┌─────────────────────────┐             │
│  │       Object Ring        │             │
│  └─────────────────────────┘             │
└─────────────────────────────────────────┘
```

### ส่วนประกอบ

| ส่วนประกอบ | หน้าที่ |
|-----------|------|
| **Proxy Server** | รับ API requests, หาตำแหน่งข้อมูลผ่าน Ring, ส่งต่อไป Storage Nodes |
| **Account Server** | เก็บรายการ Containers ใน Account |
| **Container Server** | เก็บรายการ Objects ใน Container |
| **Object Server** | เก็บ Object Data จริง |
| **Ring** | ตาราง Hash ที่กระจายข้อมูลไปยัง Nodes |

---

## The Ring — หัวใจของ Swift

**Ring** คือ Data Structure ที่กระจาย Objects ไปยัง Nodes:

```
Object Name → MD5 Hash → Partition → Replica → Storage Node
```

### ประเภท Ring

| Ring | เก็บ |
|------|------|
| **Account Ring** | ตำแหน่งของ Account Databases |
| **Container Ring** | ตำแหน่งของ Container Databases |
| **Object Ring** | ตำแหน่งของ Object Data |

### ค่าสำคัญ

| พารามิเตอร์ | คำอธิบาย |
|-----------|------|
| **Partitions** | จำนวนส่วนที่แบ่งข้อมูล (ค่าเริ่มต้น 2^18 = 262,144) |
| **Replicas** | จำนวนสำเนาของข้อมูล (ค่าเริ่มต้น 3) |
| **Devices** | ดิสก์/Partitions บน Storage Nodes |
| **Weight** | น้ำหนักสำหรับกระจายข้อมูล |

---

## Account, Container, Object

```
Account (เช่น AUTH_project-id)
└── Container (เช่น images, backups, logs)
    └── Object (เช่น photo.jpg)
        └── Metadata + Data
```

| ระดับ | คำอธิบาย |
|-------|---------|
| **Account** | คอนเทนเนอร์ระดับสูงสุด สอดคล้องกับ Project |
| **Container** | กลุ่มของ Objects (เหมือน Bucket ใน S3) |
| **Object** | ไฟล์ + Metadata แท้จริง |

---

## การใช้งาน Swift

```bash
# ตรวจสอบว่า Swift พร้อมใช้งาน
swift stat

# แสดงรายการ Containers
swift list

# สร้าง Container
swift post my-container

# อัปโหลดไฟล์
swift upload my-container report.pdf

# อัปโหลดทั้ง Directory
swift upload my-container -S 1073741824 /path/to/backups

# ดาวน์โหลดไฟล์
swift download my-container report.pdf

# แสดงรายการ Objects ใน Container
swift list my-container

# ลบ Object
swift delete my-container report.pdf

# ลบ Container ทั้งหมด
swift delete my-container
```

---

## Metadata และ Headers

```bash
# ตั้งค่า Metadata บน Container
swift post -m "X-Container-Meta-Project:webapp" my-container

# ตั้งค่า Metadata บน Object
swift post -m "X-Object-Meta-Author:alice" my-container report.pdf

# ตั้งค่า ACL (Read Access)
swift post -r ".r:*" my-container  # อนุญาตให้อ่านได้ทั้งหมด

# ตั้งค่า Expiration
swift upload -H "X-Delete-After:86400" my-container temp.log
# ไฟล์จะถูกลบอัตโนมัติหลัง 1 วัน
```

---

## Consistency และ Durability

| กลไก | คำอธิบาย |
|------|------|
| **Replication** | เก็บ 3 สำเนาของทุก Object บน Nodes ต่างกัน |
| **Quorum Read** | อ่านจาก Replica หลายตัว ยืนยันความถูกต้อง |
| **Auditor** | ตรวจสอบความสมบูรณ์ของข้อมูลเป็นระยะ |
| **Updater** | ซิงค์ Container/Account listings เมื่อ async |
| **Reaper** | จัดการ Object Expiration |

---

## สไลด์การบรรยาย

📥 [ดาวน์โหลดสไลด์การบรรยายที่ 8 (PDF)](../assets/slides/lecture08.pdf){ .md-button }

## แหล่งข้อมูลเพิ่มเติม

- [Swift Documentation](https://docs.openstack.org/swift/latest/)
- [Swift Admin Guide](https://docs.openstack.org/swift/latest/admin/)
- [S3 API Compatibility](https://docs.openstack.org/swift/latest/s3_compat.html)
