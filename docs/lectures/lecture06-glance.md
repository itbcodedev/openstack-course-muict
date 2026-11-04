# การบรรยายที่ 6: Glance — บริการ Image

## วัตถุประสงค์การเรียนรู้

- เข้าใจบทบาทของ Glance ในการจัดการ Images
- รู้จักรูปแบบ Image: QCOW2, RAW, VMDK, AMI
- สร้างและจัดการ Custom Images
- ตั้งค่า Image Properties และ Visibility

---

## Glance คืออะไร?

**Glance** เป็นบริการ Image ของ OpenStack รับผิดชอบ:

- จัดเก็บและจัดการ **Disk Images** และ **Metadata**
- Images ใช้เป็นต้นแบบสำหรับสร้าง VM (Boot Source)
- รองรับหลาย Backend Store: **File**, Swift, Ceph, HTTP
- สื่อสารผ่าน REST API (Image v2 API เป็นค่าเริ่มต้น)

---

## สถาปัตยกรรม Glance

```
┌─────────────────────────────────────────┐
│         glance-api                      │
│   (REST API สำหรับ Upload/Download)     │
└──────────────────┬──────────────────────┘
                   │
┌──────────────────▼──────────────────────┐
│         glance-registry (deprecated)      │
│         หรือ glance-api รวม Database      │
├─────────────────────────────────────────┤
│         Backend Store                     │
│   /var/lib/glance/images/ (File)          │
│   หรือ Swift / Ceph / HTTP               │
└─────────────────────────────────────────┘
```

### ส่วนประกอบ

| ส่วนประกอบ | หน้าที่ |
|-----------|------|
| **glance-api** | รับ API requests สำหรับจัดการ Image Metadata และ Binary Data |

!!! note "Glance Registry"
    `glance-registry` ถูกยกเลิกตั้งแต่ OpenStack Queens — ปัจจุบัน `glance-api` จัดการทั้งหมด

---

## Image Formats

### Disk Formats (รูปแบบไฟล์ดิสก์)

| รูปแบบ | คำอธิบาย | ใช้เมื่อไหร่ |
|--------|--------|------------|
| **QCOW2** | QEMU Copy-On-Write v2 — รองรับ Snapshot, Compression, Sparse | **ค่าเริ่มต้น** สำหรับ KVM |
| **RAW** | ไฟล์ดิบ — ไม่มี Overhead แต่ใช้พื้นที่เต็ม | สำหรับ Performance สูงสุด |
| **VMDK** | VMware Virtual Disk Format | Import จาก VMware |
| **VHD** | Microsoft Virtual Hard Disk | Hyper-V |
| **AMI** | Amazon Machine Image | AWS Migration |
| **ISO** | Optical Disk Image | ติดตั้ง OS จาก CD |

### Container Formats

| รูปแบบ | คำอธิบาย |
|--------|--------|
| **bare** | ไม่มี Container — Image ดิบ |
| **ovf** | Open Virtualization Format |
| **aki / ari / ami** | Amazon Kernel / Ramdisk / Machine Image |

---

## การจัดการ Images

```bash
# แสดงรายการ Images
openstack image list

# ดูรายละเอียด Image
openstack image show cirros

# ดาวน์โหลด Image จาก Internet
wget https://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img

# Upload Image ไปยัง Glance
openstack image create \
  --disk-format qcow2 \
  --container-format bare \
  --file cirros-0.6.2-x86_64-disk.img \
  --public \
  cirros-0.6.2

# Upload จาก URL (ไม่ต้องดาวน์โหลดก่อน)
openstack image create \
  --disk-format qcow2 \
  --container-format bare \
  --location https://example.com/image.qcow2 \
  --public \
  remote-image
```

---

## Image Visibility

```
┌─────────────────────────────────────────┐
│           Visibility                     │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐   │
│  │ Public  │ │ Shared  │ │ Private │   │
│  │ ทุกคน   │ │ แชร์ให้  │ │ เฉพาะ   │   │
│  │ ใช้ได้  │ │ Project │ │ Project │   │
│  │         │ │ ที่กำหนด│ │ ตัวเอง  │   │
│  └─────────┘ └─────────┘ └─────────┘   │
│              ┌─────────┐               │
│              │ Community│               │
│              │ ใช้ได้   │               │
│              │ แต่ไม่   │               │
│              │ Default  │               │
│              └─────────┘               │
└─────────────────────────────────────────┘
```

```bash
# เปลี่ยน Visibility
openstack image set --public my-image
openstack image set --private my-image
openstack image set --shared my-image

# แชร์ Image ให้ Project อื่น
openstack image set --shared my-image
openstack image add project my-image <project-id>
```

---

## Image Properties

Properties ช่วยให้ Nova Scheduler คัดเลือก Host ที่เหมาะสม:

```bash
# ตั้งค่า Property
openstack image set \
  --property architecture=x86_64 \
  --property hypervisor_type=kvm \
  --property hw_disk_bus=virtio \
  --property hw_vif_model=virtio \
  my-image

# ดู Properties ทั้งหมด
openstack image show my-image -c properties
```

| Property | คำอธิบาย |
|----------|---------|
| `architecture` | x86_64, arm64, i386 |
| `hypervisor_type` | kvm, xen, vmware |
| `hw_disk_bus` | virtio, scsi, ide, sata |
| `hw_vif_model` | virtio, e1000, rtl8139, ne2k_pci |
| `os_distro` | ubuntu, centos, windows |
| `os_version` | 22.04, 9, 2019 |

---

## สร้าง Custom Image

### วิธีที่ 1: ใช้ Disk Image Builder (DIB)

```bash
# ติดตั้ง Disk Image Builder
sudo dnf install -y diskimage-builder

# สร้าง Ubuntu Cloud Image
export DIB_RELEASE=jammy
disk-image-create ubuntu vm -o ubuntu-jammy

# สร้าง CentOS Stream Image
export DIB_RELEASE=stream-9
disk-image-create centos vm -o centos-stream-9
```

### วิธีที่ 2: สร้างจาก VM ที่มีอยู่ (Snapshot)

```bash
# สร้าง Snapshot จาก VM ที่รันอยู่
openstack server image create --name golden-image web-server-vm

# Snapshot จะ Capture สถานะดิสก์ปัจจุบัน
# ใช้เป็น Base Image สำหรับ Clone VMs ใหม่
```

### วิธีที่ 3: ใช้ virt-install + virt-sysprep

```bash
# สร้าง VM จาก ISO แล้วติดตั้ง OS
virt-install --name build-vm --ram 2048 --disk size=20 \
  --cdrom ubuntu.iso --network bridge=virbr0

# หลังติดตั้งเสร็จ — Sysprep เพื่อลบ ID เฉพาะเครื่อง
virt-sysprep -d build-vm

# แปลงเป็น QCOW2
qemu-img convert -O qcow2 /var/lib/libvirt/images/build-vm.qcow2 golden-image.qcow2
```

---

## Image Caching และ Copy-on-Write

```
/var/lib/nova/instances/_base/
├── golden-ubuntu.qcow2     ← Base Image (1 copy)
│
/var/lib/nova/instances/<uuid>/
└── disk                     ← CoW Layer (เฉพาะส่วนที่เปลี่ยน)
```

- **Backing File**: QCOW2 อ้างอิง Base Image ไม่ต้อง Copy ทั้งไฟล์
- **Ephemeral**: ดิสก์ VM สร้างจาก CoW — ลบ VM = ลบเฉพาะส่วนที่แตกต่าง

---

## Image ความปลอดภัย

```bash
# ลบ Image
openstack image delete my-image

# ซ่อน Image (แต่ไม่ลบ)
openstack image set --deactivate my-image

# เปิดใช้งานอีกครั้ง
openstack image set --activate my-image

# จำกัด Quota การใช้งาน Image
openstack quota set --gigabytes 500 <project-id>
```

!!! warning "ความปลอดภัย"
    อย่าใช้ Image จากแหล่งที่ไม่น่าเชื่อถือ — ตรวจสอบ Checksum ก่อน Upload

---

## สไลด์การบรรยาย

📥 [ดาวน์โหลดสไลด์การบรรยายที่ 6 (PDF)](../assets/slides/lecture06.pdf){ .md-button }

## แหล่งข้อมูลเพิ่มเติม

- [Glance Documentation](https://docs.openstack.org/glance/latest/)
- [Disk Image Builder](https://docs.openstack.org/diskimage-builder/latest/)
- [virt-sysprep Guide](https://libguestfs.org/virt-sysprep.1.html)
