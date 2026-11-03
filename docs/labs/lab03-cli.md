# การทดลองที่ 3: การใช้งาน OpenStack CLI

## วัตถุประสงค์

เชี่ยวชาญการใช้ OpenStack Command-line Client สำหรับจัดการทรัพยากร Cloud แบบ Programmatic

---

## เงื่อนไขก่อนเรียน

```bash
# โหลด credentials ของ DevStack
source /opt/stack/devstack/openrc admin

# ตรวจสอบว่า CLI ทำงานได้
openstack --version
```

---

## ส่วนที่ 1: พื้นฐาน CLI

### โครงสร้างคำสั่ง

```bash
openstack <service> <action> <resource> [options]
```

ตัวอย่าง:
```bash
openstack server list              # แสดงรายการ VMs
openstack network list             # แสดงรายการ Networks
openstack flavor list              # แสดงรายการ Flavors
openstack image list               # แสดงรายการ Images
```

### ขอความช่วยเหลือ

```bash
openstack --help                   # ความช่วยเหลือระดับบน
openstack server --help            # ความช่วยเหลือ Subcommand
openstack server create --help     # ความช่วยเหลือเฉพาะคำสั่ง
```

---

## ส่วนที่ 2: จัดการตัวตนและโครงการ

### สร้าง Project

```bash
openstack project create \
  --description "วิชา OpenStack MUICT" \
  --enable \
  muict-course
```

### สร้าง User

```bash
openstack user create \
  --password-prompt \
  --email student@muict.ac.th \
  --project muict-course \
  --enable \
  student01
```

### กำหนด Role

```bash
openstack role add \
  --user student01 \
  --project muict-course \
  member

# ตรวจสอบ
openstack role assignment list --user student01 --project muict-course
```

### สร้าง Service Account

```bash
openstack user create \
  --password servicepass \
  --project service \
  --enable \
  heat-domain-admin
```

---

## ส่วนที่ 3: การดำเนินการกับ Compute

### Flavors

```bash
# แสดงรายการ Flavors
openstack flavor list

# สร้าง Flavor แบบกำหนดเอง
openstack flavor create \
  --ram 2048 \
  --disk 20 \
  --vcpus 2 \
  --public \
  m1.student
```

### Key Pairs

```bash
# สร้าง SSH key
ssh-keygen -t rsa -f ~/.ssh/openstack_key -N ""

# อัปโหลดขึ้น OpenStack
openstack keypair create --public-key ~/.ssh/openstack_key.pub openstack_key

# แสดงรายการ Keypairs
openstack keypair list
```

### สร้าง Instance

```bash
# ตรวจสอบ Images และ Networks ที่มี
openstack image list
openstack network list

# สร้าง VM
openstack server create \
  --image cirros-0.6.2-x86_64-disk \
  --flavor m1.tiny \
  --network private \
  --key-name openstack_key \
  --security-group default \
  --wait \
  my-cli-vm

# ตรวจสอบสถานะ
openstack server show my-cli-vm
openstack server list
```

### วงจรชีวิตของ Instance

```bash
# หยุด Instance
openstack server stop my-cli-vm

# เปิด Instance
openstack server start my-cli-vm

# Reboot (soft)
openstack server reboot my-cli-vm

# Reboot (hard)
openstack server reboot --hard my-cli-vm

# Suspend / Resume
openstack server suspend my-cli-vm
openstack server resume my-cli-vm

# ลบ
openstack server delete my-cli-vm
```

### Console Access

```bash
# ขอ URL ของ VNC Console
openstack console url show --vnc my-cli-vm
```

---

## ส่วนที่ 4: การดำเนินการกับเครือข่าย

### สร้าง Network และ Subnet

```bash
# สร้าง Network
openstack network create student-cli-net

# สร้าง Subnet
openstack subnet create \
  --network student-cli-net \
  --subnet-range 10.100.0.0/24 \
  --gateway 10.100.0.1 \
  --allocation-pool start=10.100.0.10,end=10.100.0.250 \
  --dns-nameserver 8.8.8.8 \
  student-cli-subnet
```

### สร้าง Router

```bash
# สร้าง Router
openstack router create student-cli-router

# เชื่อมต่อกับ External Network
openstack router set --external-gateway public student-cli-router

# เชื่อมต่อกับ Internal Subnet
openstack router add subnet student-cli-router student-cli-subnet

# ตรวจสอบ
openstack router show student-cli-router
```

### Security Groups

```bash
# สร้าง Security Group ใหม่
openstack security group create ssh-access

# เพิ่มกฎ SSH
openstack security group rule create \
  --protocol tcp \
  --dst-port 22 \
  --remote-ip 0.0.0.0/0 \
  ssh-access

# เพิ่ม ICMP (ping)
openstack security group rule create \
  --protocol icmp \
  --remote-ip 0.0.0.0/0 \
  ssh-access

# ผูกกับ Instance
openstack server add security group my-cli-vm ssh-access
```

### Floating IPs

```bash
# จัดสรร Floating IP
openstack floating ip create public

# ดึง IP ที่จัดสรร
FLOATING_IP=$(openstack floating ip list -f value -c 'Floating IP Address' | head -1)
echo $FLOATING_IP

# ผูกกับ Instance
openstack server add floating ip my-cli-vm $FLOATING_IP

# ตรวจสอบ
openstack server show my-cli-vm -c addresses
```

---

## ส่วนที่ 5: การดำเนินการกับ Volume

```bash
# สร้าง Volume
openstack volume create --size 5 --description "Data disk" data-volume

# Attach กับ Instance
openstack server volume attach my-cli-vm data-volume

# Detach
openstack server volume detach my-cli-vm data-volume

# ลบ
openstack volume delete data-volume
```

---

## ส่วนที่ 6: Scripting ด้วย CLI

### สร้าง VMs เป็นชุด

```bash
#!/bin/bash
# batch_create.sh

NETWORK_ID=$(openstack network show private -f value -c id)
IMAGE_ID=$(openstack image show cirros-0.6.2-x86_64-disk -f value -c id)

for i in {1..3}; do
  openstack server create \
    --image $IMAGE_ID \
    --flavor m1.tiny \
    --network $NETWORK_ID \
    --key-name openstack_key \
    batch-vm-$i
done

# รอให้ทุกตัว Active
for i in {1..3}; do
  openstack server wait batch-vm-$i
done

openstack server list
```

---

!!! homework ":clipboard: การบ้าน"
    📋 **รายงานการทดลอง:**

    1. Output ของ `openstack server list` หลังจากสร้าง Instances
    2. Screenshot หรือ Output แสดง Floating IP ที่ผูกไว้
    3. Output จาก Script สร้าง VMs แบบ Batch
    4. ตอบคำถาม: `openstack server stop` ต่างจาก `openstack server suspend` อย่างไร?
    5. ตอบคำถาม: ทำไมถึงมีประโยชน์ที่จะใช้ `--wait` ใน Script อัตโนมัติ?
