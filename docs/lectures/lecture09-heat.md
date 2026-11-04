# การบรรยายที่ 9: Heat — บริการ Orchestration

## วัตถุประสงค์การเรียนรู้

- เข้าใจแนวคิด Infrastructure as Code (IaC) ใน OpenStack
- เขียน Heat Templates (HOT) ด้วย YAML
- สร้าง Stack ที่ประกอบด้วย VM, Network, Volume, Floating IP
- จัดการ Stack Lifecycle: Create, Update, Delete

---

## Heat คืออะไร?

**Heat** เป็นบริการ Orchestration ของ OpenStack:

- **Template-based** — นิยาม Infrastructure ด้วยไฟล์ YAML/JSON
- **Declarative** — บอกว่าต้องการอะไร ไม่ใช่วิธีทำ
- **Reproducible** — Deploy ซ้ำได้ด้วย Template เดียวกัน
- **Resource Management** — สร้าง VM, Network, Volume, Router, Security Groups พร้อมกัน
- **Auto-scaling** — ขยาย VM ตามเงื่อนไข (เช่น CPU > 80%)

---

## สถาปัตยกรรม Heat

```
┌─────────────────────────────────────────┐
│         heat-api                        │
│   (รับ REST API / Validate Template)   │
└──────────────────┬──────────────────────┘
                   │
┌──────────────────▼──────────────────────┐
│         heat-engine                       │
│   - Parse Template                        │
│   - เรียก APIs ของ Nova, Neutron, Cinder │
│   - ติดตามสถานะ Resources                 │
│   - จัดการ Stack Lifecycle                │
└─────────────────────────────────────────┘
```

---

## Heat Orchestration Template (HOT)

HOT ใช้ YAML นิยาม Stack:

```yaml
heat_template_version: '2021-04-16'

description: Deploy a simple web server

parameters:
  image:
    type: string
    default: cirros
    description: Image to use
  flavor:
    type: string
    default: m1.small
  key_name:
    type: string
    description: SSH keypair name

resources:
  web_server:
    type: OS::Nova::Server
    properties:
      image: { get_param: image }
      flavor: { get_param: flavor }
      key_name: { get_param: key_name }
      networks:
        - network: private-net

outputs:
  server_ip:
    description: IP address of the web server
    value: { get_attr: [web_server, first_address] }
```

---

## โครงสร้าง HOT

| ส่วน | คำอธิบาย |
|------|---------|
| `heat_template_version` | ระบุเวอร์ชันของ Template |
| `description` | คำอธิบาย Stack |
| `parameters` | ค่าที่ผู้ใช้กรอกได้ตอน Deploy |
| `resources` | Resources ที่ต้องสร้าง |
| `outputs` | ค่าที่ส่งกลับหลัง Deploy |

---

## ตัวอย่าง: 3-Tier Web Application

```yaml
heat_template_version: '2021-04-16'
description: 3-Tier Application (Web + App + DB)

parameters:
  key_name:
    type: string
  image:
    type: string
    default: ubuntu-22.04
  flavor_web:
    type: string
    default: m1.small
  flavor_db:
    type: string
    default: m1.medium

resources:
  # === Network ===
  app_network:
    type: OS::Neutron::Net
    properties:
      name: app-network

  app_subnet:
    type: OS::Neutron::Subnet
    properties:
      network: { get_resource: app_network }
      cidr: 10.0.1.0/24
      gateway: 10.0.1.1

  app_router:
    type: OS::Neutron::Router
    properties:
      external_gateway_info:
        network: public-net

  router_interface:
    type: OS::Neutron::RouterInterface
    properties:
      router: { get_resource: app_router }
      subnet: { get_resource: app_subnet }

  # === Security Group ===
  web_secgroup:
    type: OS::Neutron::SecurityGroup
    properties:
      name: web-sg
      rules:
        - protocol: tcp
          port_range_min: 80
          port_range_max: 80
          remote_ip_prefix: 0.0.0.0/0
        - protocol: tcp
          port_range_min: 22
          port_range_max: 22
          remote_ip_prefix: 0.0.0.0/0

  # === Web Server ===
  web_server:
    type: OS::Nova::Server
    properties:
      image: { get_param: image }
      flavor: { get_param: flavor_web }
      key_name: { get_param: key_name }
      networks:
        - network: { get_resource: app_network }
      security_groups:
        - { get_resource: web_secgroup }
      user_data_format: RAW
      user_data: |
        #!/bin/bash
        apt-get update
        apt-get install -y nginx
        systemctl enable nginx

  # === Database Server ===
  db_volume:
    type: OS::Cinder::Volume
    properties:
      size: 50

  db_server:
    type: OS::Nova::Server
    properties:
      image: { get_param: image }
      flavor: { get_param: flavor_db }
      key_name: { get_param: key_name }
      networks:
        - network: { get_resource: app_network }
      block_device_mapping_v2:
        - image: { get_param: image }
          delete_on_termination: false
          boot_index: 0
        - volume_id: { get_resource: db_volume }
          delete_on_termination: false
          boot_index: -1

  # === Floating IP for Web ===
  web_floating_ip:
    type: OS::Neutron::FloatingIP
    properties:
      floating_network: public-net

  web_floating_ip_assoc:
    type: OS::Neutron::FloatingIPAssociation
    properties:
      floatingip_id: { get_resource: web_floating_ip }
      port_id: { get_attr: [web_server, addresses, app_network, 0, port] }

outputs:
  web_url:
    value: { get_attr: [web_floating_ip, floating_ip_address] }
    description: Web Server IP
```

---

## Intrinsic Functions

| Function | คำอธิบาย |
|----------|---------|
| `get_param` | อ่านค่าจาก Parameter |
| `get_resource` | อ้างอิง Resource อื่น |
| `get_attr` | อ่าน Attribute ของ Resource |
| `list_join` | รวม String ด้วยตัวคั่น |
| `str_replace` | แทนที่ String ใน Template |
| `if` / `equals` | เงื่อนไข |

```yaml
user_data:
  str_replace:
    template: |
      #!/bin/bash
      echo "DB_HOST: _DB_HOST_" >> /etc/app.conf
    params:
      _DB_HOST_: { get_attr: [db_server, first_address] }
```

---

## การใช้งาน Heat CLI

```bash
# สร้าง Stack
openstack stack create -t 3tier-app.yaml --parameter key_name=my-key webapp-stack

# แสดงรายการ Stacks
openstack stack list

# ดูรายละเอียด Stack
openstack stack show webapp-stack

# ดู Events (การสร้างแต่ละ Step)
openstack stack event list webapp-stack

# อัปเดต Stack (แก้ไข Template แล้ว Deploy ใหม่)
openstack stack update -t 3tier-app-v2.yaml webapp-stack

# ลบ Stack (ลบทุก Resources ที่สร้างไว้)
openstack stack delete webapp-stack

# ดู Outputs
openstack stack output list webapp-stack
openstack stack output show webapp-stack web_url
```

---

## Auto Scaling

```yaml
resources:
  web_server_group:
    type: OS::Heat::AutoScalingGroup
    properties:
      cooldown: 60
      min_size: 1
      max_size: 5
      resource:
        type: OS::Nova::Server
        properties:
          image: ubuntu-22.04
          flavor: m1.small
          networks:
            - network: app-network

  scale_up_policy:
    type: OS::Heat::ScalingPolicy
    properties:
      adjustment_type: change_in_capacity
      scaling_adjustment: 1
      auto_scaling_group_id: { get_resource: web_server_group }

  cpu_alarm_high:
    type: OS::Ceilometer::Alarm
    properties:
      meter_name: cpu_util
      statistic: avg
      period: 600
      evaluation_periods: 1
      threshold: 80
      comparison_operator: gt
      alarm_actions:
        - { get_attr: [scale_up_policy, alarm_url] }
```

---

## สไลด์การบรรยาย

📥 [ดาวน์โหลดสไลด์การบรรยายที่ 9 (PDF)](../assets/slides/OpenStack_Heat_Architecture.pdf){ .md-button }

## แหล่งข้อมูลเพิ่มเติม

- [Heat Documentation](https://docs.openstack.org/heat/latest/)
- [HOT Template Guide](https://docs.openstack.org/heat/latest/template_guide/)
- [Heat Admin Guide](https://docs.openstack.org/heat/latest/admin/)
