# การบรรยายที่ 5: Neutron — บริการเครือข่าย

## วัตถุประสงค์การเรียนรู้

- เข้าใจสถาปัตยกรรมและส่วนประกอบของ Neutron
- อธิบายความแตกต่างระหว่าง Network Types: Local, Flat, VLAN, VXLAN, GRE
- จัดการ Networks, Subnets, Routers, Ports และ Security Groups
- ออกแบบโทโปโลยี Multi-tenant ที่ปลอดภัย

---

## Neutron คืออะไร?

**Neutron** เป็นบริการ Networking ของ OpenStack ให้บริการ:

- **Network Connectivity** — เชื่อมต่อ VMs ภายในและภายนอก
- **IP Address Management** — จัดสรร Fixed IPs, Floating IPs
- **Layer 2 Isolation** — VLAN, VXLAN, GRE สำหรับ Multi-tenancy
- **Layer 3 Services** — Routing, NAT, Load Balancing, Firewall
- **Extensible** — รองรับ Plugin และ SDN Controller

---

## สถาปัตยกรรม Neutron

```
┌─────────────────────────────────────────┐
│         neutron-server                  │
│   (REST API + ML2 Plugin + Type Driver)  │
└──────────────────┬──────────────────────┘
                   │ ผ่าน Message Queue
┌──────────────────▼──────────────────────┐
│         L2 Agent (OVS / Linux Bridge)    │
│   จัดการ Virtual Switch บนแต่ละ Host    │
├─────────────────────────────────────────┤
│         L3 Agent                         │
│   จัดการ Routing, NAT, Floating IPs       │
├─────────────────────────────────────────┤
│         DHCP Agent                       │
│   จ่าย IP Addresses (dnsmasq)            │
├─────────────────────────────────────────┤
│         Metadata Agent                   │
│   ให้บริการ Cloud-init Metadata         │
└─────────────────────────────────────────┘
```

### ส่วนประกอบหลัก

| ส่วนประกอบ | หน้าที่ |
|-----------|------|
| **neutron-server** | API Server และ Plugin (เช่น ML2) |
| **ML2 Plugin** | Modular Layer 2 — แยก Type Driver และ Mechanism Driver |
| **L2 Agent** | จัดการ Virtual Switches (OVS/Linux Bridge) บน Compute |
| **L3 Agent** | จัดการ Routers, NAT, Floating IPs |
| **DHCP Agent** | จ่าย IP ให้ VMs ผ่าน dnsmasq |
| **Metadata Agent** | ให้ 169.254.169.254 สำหรับ Cloud-init |

---

## Network Types

| ประเภท | หลักการ | ใช้เมื่อไหร่ |
|--------|--------|------------|
| **Local** | VM ติดต่อกันได้เฉพาะบน Host เดียวกัน | Testing ภายในเครื่อง |
| **Flat** | ไม่มี VLAN Tagging — ทุก VM อยู่ใน Network เดียวกัน | ง่าย แต่ไม่มี Isolation |
| **VLAN** | ใช้ 802.1Q VLAN IDs (1–4094) | ผลิตภัณฑ์ — แยก Tenant ชัดเจน |
| **VXLAN** | Overlay บน UDP Port 4789, 24-bit VNI (16 ล้าน Tenant) | ขนาดใหญ่ — ไม่ต้องปรับสวิตช์ |
| **GRE** | Generic Routing Encapsulation — Tunnel บน IP | รองรับใน Legacy |

```
┌─────────────────────────────────────────┐
│            Physical Network              │
│  ┌─────────────────────────────────┐   │
│  │  VLAN 100  (Tenant A)            │   │
│  │  VLAN 101  (Tenant B)            │   │
│  │  VXLAN VNI 5000 (Tenant C)       │   │
│  └─────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

---

## ML2 Plugin Architecture

**Modular Layer 2 (ML2)** แยกส่วนเป็น 2 ชั้น:

```
┌─────────────────────────────────────────┐
│           ML2 Plugin                    │
│  ┌──────────────┐  ┌────────────────┐  │
│  │ Type Driver  │  │ Mechanism Driver│ │
│  │ (VXLAN/VLAN) │  │ (Open vSwitch)   │ │
│  └──────────────┘  └────────────────┘  │
└─────────────────────────────────────────┘
```

| ส่วน | หน้าที่ |
|-----|------|
| **Type Driver** | กำหนดวิธี Segment Network (VLAN, VXLAN, Flat, GRE) |
| **Mechanism Driver** | กำหนดวิธีจัดการทรัพยากรจริง (OVS, Linux Bridge, L2 Population) |

---

## คำศัพท์สำคัญ

### Network
- Layer 2 Broadcast Domain ที่ VMs เชื่อมต่ออยู่
- แยกกันตาม Tenant — แต่ละ Project มี Network ของตัวเอง

### Subnet
- กำหนด IP Range, Gateway, DNS สำหรับ Network
- หนึ่ง Network มีได้หลาย Subnets

### Port
- จุดเชื่อมต่อระหว่าง VM กับ Network
- หนึ่ง Port = หนึ่ง IP Address (Fixed IP)

### Router
- เชื่อมต่อ Subnets หลายตัวเข้าด้วยกัน
- ทำ NAT (SNAT/DNAT) และ Floating IP

---

## การจัดการ Networks และ Subnets

```bash
# สร้าง Network
openstack network create --internal private-net

# สร้าง Subnet
openstack subnet create \
  --network private-net \
  --subnet-range 192.168.1.0/24 \
  --gateway 192.168.1.1 \
  --dns-nameserver 8.8.8.8 \
  private-subnet

# สร้าง External Network (Flat หรือ VLAN)
openstack network create --external --provider-network-type flat \
  --provider-physical-network physnet1 external-net

# แสดงรายการ Networks
openstack network list
openstack subnet list
```

---

## Routing และ Floating IPs

```
┌─────────────────────────────────────────┐
│         External Network                │
│     203.0.113.0/24 (Public)            │
│              │                          │
│    ┌─────────▼──────────┐             │
│    │      Router          │             │
│    │  203.0.113.10 (GW)   │             │
│    │  192.168.1.1 (INT)   │             │
│    └─────────┬───────────┘             │
│              │                          │
│    ┌─────────▼──────────┐             │
│    │   Private Network   │             │
│    │  192.168.1.0/24     │             │
│    │  VM: 192.168.1.5    │             │
│    └────────────────────┘             │
└─────────────────────────────────────────┘
```

```bash
# สร้าง Router
openstack router create my-router

# เชื่อมต่อ Router กับ Subnet (Internal)
openstack router add subnet my-router private-subnet

# ตั้งค่า External Gateway
openstack router set --external-gateway external-net my-router

# จัดสรร Floating IP
openstack floating ip create external-net

# ผูก Floating IP กับ VM
openstack server add floating ip my-instance 203.0.113.15
```

---

## Security Groups

**Security Group** เป็น Stateful Firewall ระดับ Port:

```bash
# แสดงรายการ Security Groups
openstack security group list

# สร้าง Security Group
openstack security group create web-servers

# เพิ่ม Rule: อนุญาต SSH
openstack security group rule create \
  --protocol tcp --dst-port 22 --remote-ip 0.0.0.0/0 \
  web-servers

# เพิ่ม Rule: อนุญาต HTTP + HTTPS
openstack security group rule create \
  --protocol tcp --dst-port 80 --remote-ip 0.0.0.0/0 \
  web-servers
openstack security group rule create \
  --protocol tcp --dst-port 443 --remote-ip 0.0.0.0/0 \
  web-servers

# ผูก Security Group กับ VM
openstack server add security group my-instance web-servers
```

!!! info "Default Security Group"
    เริ่มต้น Security Group จะ **ปิดทุก Ingress** และ **เปิดทุก Egress** — ต้องเพิ่ม Rule เอง

---

## Self-Service Network แบบสมบูรณ์

ขั้นตอนสร้าง Network ที่ VMs ออกอินเทอร์เน็ตได้:

```bash
# 1. สร้าง Private Network + Subnet
openstack network create private-net
openstack subnet create --network private-net \
  --subnet-range 10.0.0.0/24 --gateway 10.0.0.1 private-subnet

# 2. สร้าง Router + Connect
openstack router create router1
openstack router add subnet router1 private-subnet
openstack router set --external-gateway public-net router1

# 3. สร้าง VM
openstack server create --image cirros --flavor m1.tiny \
  --network private-net --key-name my-key vm1

# 4. จัดสรร Floating IP และผูก
FLOATING_IP=$(openstack floating ip create public-net -c floating_ip_address -f value)
openstack server add floating ip vm1 $FLOATING_IP
```

---

## สไลด์การบรรยาย

📥 [ดาวน์โหลดสไลด์การบรรยายที่ 5 (PDF)](../assets/slides/OpenStack_Neutron_Architecture.pdf){ .md-button }

## แหล่งข้อมูลเพิ่มเติม

- [Neutron Documentation](https://docs.openstack.org/neutron/latest/)
- [Neutron Document pdf](../assets/slides/doc-neutron.pdf)
- [Open vSwitch](https://www.openvswitch.org/)
- [Neutron Networking Guide](https://docs.openstack.org/neutron/latest/admin/)
