นี่คือ `README.md` ฉบับสมบูรณ์ที่ผมเขียนให้ครับ โดยรวบรวมรายละเอียดทั้งหมดที่เราคุยกันมา อธิบายกลไกของ Bug แต่ละตัว (Root Cause) เพื่อให้คุณอ่านทำความเข้าใจก่อน (ตามแผนที่คุณวางไว้) และมี Reference ให้ศึกษาต่อ

คุณสามารถก๊อปปี้ข้อความนี้ไปบันทึกเป็นไฟล์ `README.md` ไว้ในโฟลเดอร์ `centos9-patches/` ได้เลยครับ

---

```markdown
# DevStack on CentOS Stream 9: The 10 Bug Fixes & Automation Scripts

Repository นี้รวบรวมชุดสคริปต์ (Patch & Automation) สำหรับการติดตั้ง **OpenStack DevStack** บน **CentOS Stream 9** 

โดยปกติแล้ว DevStack ถูกออกแบบและทดสอบบน Ubuntu เป็นหลัก (First-class support) การนำไปรันบน CentOS Stream 9 (RHEL-based) จึงมักพบปัญหาเรื่องความเข้ากันได้ (Compatibility) ของ Package เวอร์ชัน, Systemd, และ Timing ของ Service ต่างๆ

สคริปต์ชุดนี้ถูกออกแบบมาให้แก้ปัญหาเหล่านั้นโดยอัตโนมัติ (Idempotent) เพื่อให้ CentOS Stream 9 ทำงานได้เทียบเท่า Ubuntu Baseline

---

## 🚀 วิธีการใช้งาน (Quick Start)

### 1. สิ่งที่ต้องเตรียม (Prerequisites)
- ติดตั้ง CentOS Stream 9 (แนะนำเวอร์ชัน Minimal)
- ตั้งค่า Network ให้มี Interface อย่างน้อย 2 ตัว (เช่น `eth0` สำหรับ Management และ `eth1` สำหรับ External/Provider)
- ตรวจสอบให้แน่ใจว่าเครื่องสามารถออกอินเทอร์เน็ตได้ (สำหรับดาวน์โหลด Package และ Image)
- **ห้ามรันด้วย `root` เด็ดขาด!** ต้องสร้าง User `stack` และให้สิทธิ์ `sudo` ก่อน

### 2. ขั้นตอนการติดตั้ง (ครั้งแรก)
```bash
# 1. Clone DevStack
git clone https://opendev.org/openstack/devstack /opt/stack/devstack
cd /opt/stack/devstack

# 2. สร้างไฟล์ local.conf (ตัวอย่าง)
cat > local.conf << 'EOF'
[[local|localrc]]
ADMIN_PASSWORD=Muict2024
DATABASE_PASSWORD=$ADMIN_PASSWORD
RABBIT_PASSWORD=$ADMIN_PASSWORD
SERVICE_PASSWORD=$ADMIN_PASSWORD

HOST_IP=10.0.0.10

enable_service rabbit mysql key
enable_service n-api n-cpu n-cond n-sch n-novnc n-cauth
enable_service g-api g-reg
enable_service cinder c-sch c-api c-vol
enable_service neutron q-svc q-ovn-metadata-agent
enable_service s-proxy s-object s-container s-account
enable_service placement-api

# Network Config (ปรับตาม Interface จริงของคุณ)
FLOATING_RANGE=192.168.56.0/24
PUBLIC_BRIDGE=br-ex
FLAT_INTERFACE=eth2
PUBLIC_INTERFACE=eth2
FIXED_RANGE=10.0.0.0/24
FIXED_NETWORK_SIZE=256
Q_FLOATING_ALLOCATION_POOL=start=192.168.56.100,end=192.168.56.200

disable_service tempest
LOGFILE=/opt/stack/logs/stack.sh.log
LOG_COLOR=True
VERBOSE=True
EOF

# 3. คัดลอกสคริปต์ชุดนี้ไปไว้ใน DevStack Root
cp -r centos9-patches /opt/stack/devstack/

# 4. เข้าไปที่โฟลเดอร์สคริปต์และรัน
cd /opt/stack/devstack/centos9-patches
./bootstrap-all.sh
```

### 3. การใช้งานครั้งต่อไป (หลัง Reboot หรือต้องการ Start ใหม่)
```bash
cd /opt/stack/devstack/centos9-patches

# ตรวจสอบและแก้ไข Network (เช่น default route ผ่าน br-ex)
./05-fix-network.sh

# ตรวจสอบความพร้อมก่อนรัน
./30-pre-stack-check.sh

# รัน stack.sh พร้อมระบบจัดการ Service อัตโนมัติ
./40-run-stack.sh
```

---

## 🐛 10 ปัญหาที่พบและวิธีแก้ (Root Cause Analysis)

นี่คือรายละเอียดของ Bug ทั้ง 10 ตัวที่สคริปต์ชุดนี้เข้าไปจัดการ หากต้องการศึกษาเชิงลึก แนะนำให้อ่าน Reference ที่แนบมา

### 1. Python 3.9 เก่าเกินไป (OpenStack 2026.1+ ต้องการ 3.10 ขึ้นไป)
- **อาการ:** `ERROR: Package 'openstack-requirements' requires a different Python: 3.9.25 not in '>=3.11'`
- **สาเหตุ:** OpenStack ตัดการรองรับ Python 3.9 แล้ว แต่ CentOS Stream 9 มาพร้อม Python 3.9 เป็นค่าเริ่มต้น
- **การแก้ไข:** ติดตั้ง Python 3.11 และบังคับให้ DevStack ใช้ผ่านตัวแปร `PYTHON3_VERSION=3.11` ใน `stackrc` และ `local.conf`
- **ไฟล์ที่แก้:** `stackrc`, `local.conf`
- **อ้างอิง:** [OpenStack Python Requirements](https://governance.openstack.org/tc/reference/runtimes/2026.1.html)

### 2. Dependency Conflict ใน `upper-constraints.txt`
- **อาการ:** `ResolutionImpossible: for help visit...` (เช่น `testtools`, `fixtures`, `requests`, `etcd3gw`)
- **สาเหตุ:** ไฟล์ Constraints ของ DevStack บังคับเวอร์ชันแบบ `===` เข้มงวดเกินไป จนชนกับความต้องการของ Package ลูก
- **การแก้ไข:** Comment บรรทัดที่ชนใน `/opt/stack/requirements/upper-constraints.txt`
- **ไฟล์ที่แก้:** `/opt/stack/requirements/upper-constraints.txt`
- **อ้างอิง:** [OpenStack Requirements](https://opendev.org/openstack/requirements)

### 3. RabbitMQ ไม่มีใน Repo ของ CentOS Stream 9
- **อาการ:** `Failed to enable unit: Unit file rabbitmq-server.service does not exist.`
- **สาเหตุ:** Repo เริ่มต้นของ CentOS Stream 9 ไม่มี RabbitMQ ต้องใช้ Repo จาก NFV SIG หรือ Official RabbitMQ
- **การแก้ไข:** ติดตั้ง RabbitMQ ผ่าน RPM อย่างเป็นทางการ และ Patch `lib/rpc_backend` เพื่อไม่ให้ DevStack พยายามถอน/ติดตั้งทับ
- **ไฟล์ที่แก้:** `lib/rpc_backend`
- **อ้างอิง:** [RabbitMQ RPM on CentOS](https://www.rabbitmq.com/install-rpm.html)

### 4. ปัญหา `getent hosts` คืนค่า IPv6 (fe80::)
- **อาการ:** RabbitMQ ไม่ยอม Start หรือ `rabbitmqctl` ค้าง เพราะ resolve hostname ผิด
- **สาเหตุ:** `getent hosts` ของ Erlang/RabbitMQ จะตรวจ IPv6 ก่อน ถ้าใน `/etc/hosts` ไม่มี IPv6 ระบุไว้ จะไปดึง IPv6 Link-Local (`fe80::`) จาก Network Interface มาใช้
- **การแก้ไข:** เพิ่ม `linux.local` (ชื่อ Hostname) ทั้งในบรรทัด IPv4 และ IPv6 ใน `/etc/hosts`
- **ไฟล์ที่แก้:** `/etc/hosts`
- **อ้างอิง:** [RabbitMQ Hostname Resolution](https://www.rabbitmq.com/docs/clustering#hostname-resolution)

### 5. `rsyncd` ไม่มีใน Package `rsync`
- **อาการ:** `Failed to start rsyncd.service: Unit rsyncd.service not found.`
- **สาเหตุ:** CentOS/RHEL แยก Daemon ออกเป็น Package `rsync-daemon` แต่ DevStack ติดตั้งแค่ `rsync`
- **การแก้ไข:** ติดตั้ง `rsync-daemon`
- **ไฟล์ที่แก้:** `00-prereqs.sh`

### 6. OVS/OVN Unit File ชื่อต่างจากที่ DevStack คาดหวัง
- **อาการ:** `Failed to enable unit: Unit file ovsdb-server.service does not exist.`
- **สาเหตุ:** แพ็กเกจ NFV SIG ใช้ `openvswitch.service` ตัวเดียวจัดการทั้งหมด ไม่ได้แยกเป็น `ovsdb-server` และ `ovs-vswitchd` แบบ Ubuntu
- **การแก้ไข:** Patch `lib/neutron_plugins/ovn_agent` ให้เรียกใช้ `openvswitch.service`
- **ไฟล์ที่แก้:** `lib/neutron_plugins/ovn_agent`
- **อ้างอิง:** [OVS Systemd Units](https://docs.openvswitch.org/en/latest/intro/install/fedora/)

### 7. OVN ไม่มี Unit File และไม่มี Schema
- **อาการ:** `Unit file ovn-central.service does not exist.`
- **สาเหตุ:** แพ็กเกจ `ovn24.09` ไม่ได้ติดตั้ง Systemd Unit หรือ Schema File มาให้ครบ
- **การแก้ไข:** สร้าง `ovn-central.service` เอง และติดตั้ง `ovn24.09-central` เพื่อเอา Schema มาสร้าง Database (`ovsdb-tool create`)
- **ไฟล์ที่แก้:** `20-setup-systemd.sh`

### 8. Neutron Geneve Pool = 0 (Race Condition)
- **อาการ:** `503 Server Error for url: .../networks, Unable to create the network. No project network is available for allocation.`
- **สาเหตุ:** Neutron มี Bug เรื่อง Timing ทำให้บางครั้ง Worker ไม่ยอม Initialize `ml2_geneve_allocations` ลง Database (Pool ว่างเปล่า)
- **การแก้ไข:** ตรวจสอบ Pool และ Restart `neutron-api` เพื่อบังคับให้ Initialize ใหม่
- **ไฟล์ที่แก้:** `30-pre-stack-check.sh`, `40-run-stack.sh`
- **อ้างอิง:** [Launchpad Bug #2033095](https://bugs.launchpad.net/neutron/+bug/2033095)

### 9. Nova Compute ไม่ลงทะเบียน (RabbitMQ 4.3+ Breaking Change)
- **อาการ:** `Didn't find service registered by hostname after 60 seconds` และ `openstack compute service list` ว่างเปล่า
- **สาเหตุ:** RabbitMQ 4.3 ขึ้นไปปิดการใช้งาน `transient_nonexcl_queues` เป็นค่าเริ่มต้น ทำให้ `oslo.messaging` (ที่ Nova ใช้) ไม่สามารถสร้าง Reply Queue สำหรับ RPC ได้
- **การแก้ไข:** เพิ่ม `deprecated_features.permit.transient_nonexcl_queues = true` ใน `/etc/rabbitmq/rabbitmq.conf`
- **ไฟล์ที่แก้:** `20-setup-systemd.sh`
- **อ้างอิง:** [RabbitMQ 4.3 Deprecations](https://www.rabbitmq.com/release-information#rabbitmq-4.3.0), [oslo.messaging Quorum Queues](https://opendev.org/openstack/oslo.messaging)

### 10. Horizon Login ไม่ได้หลัง Reboot (Connection Refused)
- **อาการ:** `ConnectionRefusedError: [Errno 111] Connection refused` ที่หน้า Login
- **สาเหตุ:** `memcached.service` ไม่ถูก `enable` ให้เริ่มทำงานอัตโนมัติหลัง Reboot
- **การแก้ไข:** `systemctl enable --now memcached`
- **ไฟล์ที่แก้:** `00-prereqs.sh`, `40-run-stack.sh`

---

## 📂 โครงสร้างไฟล์สคริปต์ (Directory Structure)

```text
centos9-patches/
├── 00-prereqs.sh           # ติดตั้ง Package พื้นฐาน, Python 3.11, RabbitMQ, OVS/OVN, Memcached
├── 05-fix-network.sh       # ตรวจสอบและลบ Default Route ที่ผิดพลาดผ่าน br-ex
├── 10-patch-devstack.sh    # แก้ไข Source Code ของ DevStack (stackrc, lib/*) และ Apache Timeout
├── 20-setup-systemd.sh     # สร้าง Systemd Unit สำหรับ OVN, Database, และแก้ RabbitMQ Config
├── 30-pre-stack-check.sh   # ตรวจสอบความพร้อมของ Service ก่อนรัน stack.sh
├── 40-run-stack.sh         # รัน stack.sh พร้อมดักจับ Log และ Restart Service ที่มีปัญหา Timing
├── bootstrap-all.sh        # รันสคริปต์ทั้งหมดตามลำดับ (สำหรับการติดตั้งครั้งแรก)
└── README.md               # เอกสารนี้
```

---

## 📚 แหล่งอ้างอิงสำหรับศึกษาต่อ (References)
- [DevStack Official Documentation](https://docs.openstack.org/devstack/latest/)
- [OpenStack Neutron OVN Driver](https://docs.openstack.org/neutron/latest/admin/ovn/index.html)
- [Open vSwitch on Fedora/CentOS](https://docs.openvswitch.org/en/latest/intro/install/fedora/)
- [RabbitMQ Configuration](https://www.rabbitmq.com/docs/configure)

---

## 📝 License
สคริปต์ชุดนี้แจกจ่ายภายใต้ License แบบ MIT ท่านสามารถนำไปใช้ แก้ไข และแชร์ต่อได้อย่างอิสระ
```

---

### 💡 คำแนะนำเพิ่มเติมสำหรับคุณ

1. **อ่านทำความเข้าใจก่อน:** ตามแผนของคุณ ให้ลองไล่อ่าน Bug ทั้ง 10 ข้อใน README นี้ แล้วเปิดโค้ด `.sh` ดูว่ามันแก้ตรงไหน (เช่น `sed` แก้ไฟล์ไหน) ถ้าตรงไหนไม่เข้าใจ ให้ถามผมได้เลยครับ
2. **ลองใช้จริง:** รัน `bootstrap-all.sh` อีกครั้งใน VM เปล่า (ถ้ามี) เพื่อดูว่ามันเวิร์กจริงไหม
3. **ปรับแต่งให้เข้ากับตัวเอง:** คุณอาจจะเพิ่ม IP, Password, หรือ Interface ใน `local.conf` ให้ตรงกับสภาพแวดล้อมจริงของคุณ

ถ้าคุณพร้อมที่จะแชร์ให้ชุมชนแล้ว (เช่นจะเอาไปลง GitHub) บอกผมได้นะครับ ผมช่วยแปลหรือปรับภาษาให้เป็นทางการเพิ่มเติมได้ครับ 🚀