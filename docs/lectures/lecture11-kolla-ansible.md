# การบรรยายที่ 11: การติดตั้งระดับ Production ด้วย Kolla-Ansible

## วัตถุประสงค์การเรียนรู้

- เข้าใจความแตกต่างระหว่าง DevStack และ Production Deployment
- รู้จัก Kolla-Ansible และ Containerized OpenStack
- อธิบายสถาปัตยกรรม Production: HA, Load Balancing, TLS
- เตรียม Inventory และ Deploy OpenStack ด้วย Ansible Playbooks

---

## DevStack vs Production

| ด้าน | DevStack | Production |
|------|----------|-----------|
| **วัตถุประสงค์** | พัฒนา/ทดสอบ | ใช้งานจริง |
| **Node** | 1 Node | Multi-node |
| **HA** | ไม่มี | มี (Redundant Controllers) |
| **Storage** | Loopback/LVM | Ceph/NFS/iSCSI |
| **Network** | Single Flat | VLAN/VXLAN + External |
| **Security** | ไม่มี TLS | TLS ทุก Service |
| **Deployment** | Shell Script | Ansible/Kolla/Puppet |

---

## Kolla-Ansible คืออะไร?

**Kolla-Ansible** เป็นเครื่องมือ Deploy OpenStack ใน Docker Containers:

- **Containerized** — แต่ละ Service รันใน Docker Container
- **Immutable** — อัปเกรดโดยเปลี่ยน Image ไม่ต้องแก้ไข Server
- **Production Ready** — รองรับ HA, TLS, Rolling Upgrades
- **Ansible-based** — ใช้ Ansible Playbooks จัดการทั้งหมด
- **Pre-built Images** — มี Official Images จาก OpenStack

---

## สถาปัตยกรรม Production

```
┌─────────────────────────────────────────┐
│         Load Balancer (HAProxy)          │
│         Keepalived (VIP)                  │
└──────────────────┬──────────────────────┘
                   │
┌──────────────────▼──────────────────────┐
│  Controller 1    │  Controller 2        │
│  MariaDB (Galera)│  MariaDB (Galera)    │
│  RabbitMQ       │  RabbitMQ            │
│  Keystone API   │  Keystone API        │
│  Nova API       │  Nova API            │
│  Neutron Server │  Neutron Server      │
│  Horizon        │  Horizon             │
│  Memcached      │  Memcached           │
└─────────────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────┐
│  Compute Node 1  │  Compute Node 2     │
│  Nova Compute     │  Nova Compute        │
│  Neutron L2 Agent │ Neutron L2 Agent     │
│  Neutron L3 Agent │ Neutron L3 Agent     │
│  Cinder Volume    │ Cinder Volume        │
│  (KVM + OVS)      │ (KVM + OVS)          │
└─────────────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────┐
│  Storage Node (Ceph / NFS)             │
│  OSD 1, OSD 2, OSD 3                   │
│  Cinder Backend / Glance Backend        │
└─────────────────────────────────────────┘
```

---

## ส่วนประกอบ Kolla

| ส่วนประกอบ | หน้าที่ |
|-----------|------|
| **kolla-build** | สร้าง Docker Images |
| **kolla-ansible** | Deploy และ Manage Containers |
| **kolla-ansible inventory** | กำหนดบทบาทของแต่ละ Node |
| **globals.yml** | ตั้งค่าทั่วไปของ Deployment |
| **passwords.yml** | เก็บรหัสผ่านทั้งหมด |

---

## ขั้นตอน Deploy ด้วย Kolla-Ansible

### 1. ติดตั้งสภาพแวดล้อม

```bash
# ติดตั้ง Dependencies
sudo dnf install -y python3-pip git
pip3 install --user kolla-ansible

# สร้าง Directory
mkdir -p /etc/kolla
cp -r ~/.local/share/kolla-ansible/etc_examples/kolla/* /etc/kolla/
cp ~/.local/share/kolla-ansible/ansible/inventory/* .
```

### 2. กำหนด Inventory

```ini
# multinode inventory
[control]
ctrl01 ansible_host=10.0.0.11
ctrl02 ansible_host=10.0.0.12
ctrl03 ansible_host=10.0.0.13

[network]
ctrl01 ansible_host=10.0.0.11
ctrl02 ansible_host=10.0.0.12
ctrl03 ansible_host=10.0.0.13

[compute]
comp01 ansible_host=10.0.0.21
comp02 ansible_host=10.0.0.22

[storage]
stg01 ansible_host=10.0.0.31

[mariadb:children]
control

[rabbitmq:children]
control

[keystone:children]
control

[glance:children]
control

[nova:children]
control

[neutron:children]
network

[cinder:children]
control

[horizon:children]
control
```

### 3. ตั้งค่า globals.yml

```yaml
# /etc/kolla/globals.yml
kolla_base_distro: "centos"
kolla_install_type: "source"
openstack_release: "2024.1"

# Network
kolla_internal_vip_address: "10.0.0.10"
network_interface: "eth0"
neutron_external_interface: "eth1"

# Storage
enable_cinder: "yes"
cinder_backend_lvm: "yes"
enable_ceph: "no"

# Services
enable_haproxy: "yes"
enable_keepalived: "yes"
enable_memcached: "yes"
enable_rabbitmq: "yes"

# TLS (Production)
enable_tls: "yes"
kolla_enable_tls: "yes"
```

### 4. สร้างรหัสผ่าน

```bash
kolla-genpwd
# แก้ไข passwords.yml ถ้าต้องการกำหนดเอง
```

### 5. Bootstrap และ Deploy

```bash
# ตรวจสอบ Inventory
ansible -i multinode all -m ping

# Bootstrap Servers (ติดตั้ง Docker, NTP, etc.)
kolla-ansible -i multinode bootstrap-servers

# Prechecks
kolla-ansible -i multinode prechecks

# Deploy OpenStack
kolla-ansible -i multinode deploy

# Post-deploy (สร้าง Keystone Endpoints, OpenRC)
kolla-ansible -i multinode post-deploy
```

### 6. ใช้งาน OpenStack

```bash
# โหลด OpenRC
source /etc/kolla/admin-openrc.sh

# ตรวจสอบ Services
openstack service list
openstack endpoint list
openstack compute service list
openstack network agent list
```

---

## การ Upgrade

```bash
# 1. เปลี่ยน openstack_release ใน globals.yml
openstack_release: "2024.2"

# 2. Pull Images ใหม่
kolla-ansible -i multinode pull

# 3. Rolling Upgrade
kolla-ansible -i multinode upgrade
```

---

## สไลด์การบรรยาย

📥 [ดาวน์โหลดสไลด์การบรรยายที่ 11 (PDF)](../assets/slides/lecture11.pdf){ .md-button }

## แหล่งข้อมูลเพิ่มเติม

- [Kolla-Ansible Documentation](https://docs.openstack.org/kolla-ansible/latest/)
- [Kolla Quick Start](https://docs.openstack.org/kolla-ansible/latest/user/quickstart.html)
- [OpenStack HA Guide](https://docs.openstack.org/ha-guide/latest/)
