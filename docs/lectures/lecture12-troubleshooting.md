# การบรรยายที่ 12: การแก้ไขปัญหาและแนวปฏิบัติที่ดี

## วัตถุประสงค์การเรียนรู้

- รู้จับเครื่องมือและเทคนิคการ Debug OpenStack
- ระบุปัญหาทั่วไปในแต่ละบริการและวิธีแก้ไข
- เข้าใจ Log Files, Message Queue และ Database
- ปฏิบัติตามแนวทางที่ดีสำหรับ Production

---

## แนวคิดการ Troubleshoot

```
1. Identify     → ระบุว่าปัญหาคืออะไร เกิดที่ Service ไหน
2. Isolate      → แยก Scope ของปัญหา
3. Reproduce    → สร้างสถานการณ์ซ้ำได้หรือไม่
4. Gather Data  → เก็บ Logs, Metrics, Configuration
5. Analyze      → วิเคราะห์หาสาเหตุ
6. Fix & Verify → แก้ไขและยืนยันว่าปัญหาหาย
```

---

## เครื่องมือหลักในการ Troubleshoot

### Logs

| ไฟล์ Log | บริการ |
|----------|--------|
| `/var/log/nova/*.log` | Nova |
| `/var/log/neutron/*.log` | Neutron |
| `/var/log/keystone/*.log` | Keystone |
| `/var/log/glance/*.log` | Glance |
| `/var/log/cinder/*.log` | Cinder |
| `/var/log/apache2/` หรือ `/var/log/httpd/` | WSGI / Horizon |
| `/var/log/rabbitmq/` | Message Queue |
| `/var/log/mariadb/` | Database |
| `/var/log/kolla/` | Kolla Containers |

```bash
# ดู Log แบบ Real-time
tail -f /var/log/nova/nova-compute.log

# ค้นหา Error
sudo grep -i error /var/log/nova/*.log

# ค้นหา Traceback
sudo grep -A 20 "Traceback" /var/log/neutron/neutron-server.log

# ดู Log ของ Container (Kolla)
docker logs nova_compute
docker logs --tail 100 neutron_server
```

### CLI Debugging

```bash
# ตรวจสอบสถานะ Services
openstack compute service list
openstack network agent list
openstack volume service list

# ตรวจสอบ Endpoints
openstack endpoint list

# ตรวจสอบ Token และ Authentication
openstack token issue

# ดูรายละเอียด VM
openstack server show <instance-id>

# ดู Console Log ของ VM
openstack console log show <instance-id>
```

---

## ปัญหาทั่วไปและวิธีแก้ไข

### 1. VM ไม่สามารถ Boot ได้

| อาการ | สาเหตุที่เป็นไปได้ | วิธีแก้ |
|-------|-------------------|---------|
| No valid host found | Scheduler ไม่พบ Host ที่เหมาะสม | ตรวจสอบ Resources (RAM, CPU), Flavor, Availability Zone |
| Image not found | Glance Image ไม่มีหรือถูกลบ | ตรวจสอบ `openstack image list` |
| Volume not found | Boot Volume หาย | ตรวจสอบ `openstack volume list` |
| Build timeout | Network/DHCP มีปัญหา | ตรวจสอบ Neutron DHCP Agent |

```bash
# ตรวจสอบ Scheduler Filter
openstack hypervisor list
openstack hypervisor show <hypervisor-id>
# ดู nova-scheduler.log หาสาเหตุที่ Host ถูก Reject
```

### 2. Network ไม่ทำงาน

| อาการ | สาเหตุ | วิธีแก้ |
|-------|--------|---------|
| VM ไม่ได้ IP | DHCP Agent ล่ม | `systemctl status neutron-dhcp-agent` |
| ไม่สามารถ Ping ออก | Router/Gateway ผิด | ตรวจสอบ Router และ External Gateway |
| Security Group บล็อก | Rules ไม่ถูกต้อง | `openstack security group rule list` |
| OVS Bridge ผิด | br-int หรือ br-ex ไม่มี | `ovs-vsctl show` |

```bash
# ตรวจสอบ OVS
sudo ovs-vsctl show
sudo ovs-ofctl dump-flows br-int

# ตรวจสอบ Namespace
sudo ip netns list
sudo ip netns exec qrouter-<id> ip addr

# ตรวจสอบ DHCP
sudo ip netns exec qdhcp-<id> ps aux | grep dnsmasq
```

### 3. Keystone Authentication ล้มเหลว

```bash
# ตรวจสอบ Keystone Service
systemctl status openstack-keystone

# ตรวจสอบ Database Connection
sudo grep -i "database connection" /etc/keystone/keystone.conf

# ตรวจสอบ Endpoints
openstack endpoint list --service identity

# ทดสอบ Token
openstack --os-auth-url http://localhost:5000/v3 token issue
```

### 4. Cinder Volume ผิดพลาด

| อาการ | สาเหตุ | วิธีแก้ |
|-------|--------|---------|
| Volume สร้างไม่ได้ | Backend Storage เต็ม | ตรวจสอบพื้นที่ว่าง |
| Attach ไม่ได้ | VM State ผิด | ตรวจสอบ VM ต้องเป็น Active |
| Detach ล้มเหลว | Device ถูกใช้งานอยู่ | Unmount ภายใน VM ก่อน |

```bash
# ตรวจสอบ Volume Status
openstack volume list
openstack volume show <volume-id>

# ตรวจสอบ LVM (ถ้าใช้ LVM Backend)
sudo lvs
sudo vgs
sudo pvs
```

---

## Message Queue และ Database

### RabbitMQ

```bash
# ตรวจสอบสถานะ RabbitMQ
sudo rabbitmqctl status
sudo rabbitmqctl list_queues
sudo rabbitmqctl list_connections

# ลบ Queue ที่ค้าง (ถ้ามีปัญหา)
sudo rabbitmqctl purge_queue notifications.info
```

### MariaDB

```bash
# เข้าสู่ Database
mysql -u root -p -e "SHOW PROCESSLIST;"

# ตรวจสอบ Tables
mysql -u root -p -e "USE nova; SHOW TABLES;"

# ตรวจสอบ Galera Cluster (ถ้าใช้ HA)
mysql -u root -p -e "SHOW STATUS LIKE 'wsrep%';"
```

---

## แนวปฏิบัติที่ดี (Best Practices)

### Security

- ✅ เปิดใช้งาน **TLS** ทุก API Endpoint
- ✅ ใช้ **Strong Passwords** และหมุนเวียนรหัสผ่านเป็นระยะ
- ✅ จำกัด **Security Groups** ให้แคบที่สุด
- ✅ ปิด **Unused Services**
- ✅ ใช้ **RBAC** — ให้สิทธิ์ตามบทบาท
- ✅ อัปเดต **Security Patches** อย่างสม่ำเสมอ

### Operations

- ✅ สำรอง **Database** ทุกวัน
- ✅ เก็บ **Log** อย่างน้อย 30 วัน
- ✅ ตั้งค่า **Monitoring** (Ceilometer + Alarms)
- ✅ ใช้ **Configuration Management** (Ansible/Puppet)
- ✅ ทดสอบ **Disaster Recovery** เป็นระยะ
- ✅ จัดทำ **Runbooks** สำหรับ Incidents ที่พบบ่อย

### Performance

- ✅ ใช้ **SSD** สำหรับ Database และ Boot Disks
- ✅ แยก **Control Plane** และ **Data Plane** Network
- ✅ ใช้ **Jumbo Frames** (MTU 9000) สำหรับ Storage Network
- ✅ กำหนด **CPU Pinning** สำหรับ VMs ที่ต้องการ Performance สูง

---

## Scenarios การ Debug

### Scenario 1: VM สร้างแล้วแต่ไม่มี IP
```bash
# Step 1: ตรวจสอบ Neutron Port
openstack port list --server <instance-id>

# Step 2: ตรวจสอบ DHCP Agent
openstack network agent list

# Step 3: ตรวจสอบ dnsmasq ใน qdhcp namespace
sudo ip netns exec qdhcp-<net-id> cat /var/lib/neutron/dhcp/<net-id>/host

# Step 4: ดู Console Log
openstack console log show <instance-id>
# ดูว่า Cloud-init ได้รับ Metadata หรือไม่
```

### Scenario 2: API ตอบช้า
```bash
# Step 1: ตรวจสอบ Load
htop / iostat / vmstat

# Step 2: ตรวจสอบ Database Connections
mysql -e "SHOW PROCESSLIST;" | wc -l

# Step 3: ตรวจสอบ Message Queue
sudo rabbitmqctl list_queues | grep -i "nova"

# Step 4: ตรวจสอบ API Logs หา Slow Queries
```

---

## สไลด์การบรรยาย

📥 [ดาวน์โหลดสไลด์การบรรยายที่ 12 (PDF)](../assets/slides/Troubleshoot_OpenStack.pdf){ .md-button }

## แหล่งข้อมูลเพิ่มเติม

- [OpenStack Troubleshooting Guide](https://docs.openstack.org/ops-guide/)
- [OpenStack Log Analysis](https://docs.openstack.org/ops-guide/ops-logging_monitoring.html)
- [RabbitMQ Management](https://www.rabbitmq.com/management.html)
- [MariaDB Galera](https://mariadb.com/kb/en/galera-cluster/)
