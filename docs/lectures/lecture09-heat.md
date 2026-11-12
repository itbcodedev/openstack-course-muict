---
title: "การบรรยายที่ 9: Heat — บริการ Orchestration"
---

# การบรรยายที่ 9: Heat — บริการ Orchestration

## วัตถุประสงค์การเรียนรู้

- อธิบายความต่างระหว่าง Template, Stack และ Resource
- เขียน Heat Orchestration Template (HOT) ที่ให้ VM ใช้ Port บน Network ที่มีอยู่
- สร้าง อัปเดต และลบ Stack แล้วตรวจผลด้วย Heat, Nova และ Neutron

## Heat ทำอะไร?

ใน [บท Nova](lecture04-nova.md) และ [บท Neutron](lecture05-neutron.md) เราสร้าง VM และตรวจพอร์ตทีละขั้น เมื่ออยากสร้างชุดทรัพยากรแบบเดิมซ้ำ ให้เก็บคำอธิบายไว้ใน **Template** แล้วส่งให้ **Heat** ประสานงานกับบริการที่เกี่ยวข้อง แนวทางนี้เรียกว่า **Infrastructure as Code (IaC)** เพราะเราเก็บโครงสร้างไว้ในไฟล์ที่อ่าน ทบทวน และติดตามการเปลี่ยนแปลงได้

| คำศัพท์ | ตัวอย่างในบทนี้ |
|---|---|
| **Template** | ไฟล์ YAML ที่ระบุว่าต้องการ Port หนึ่งรายการและ VM หนึ่งเครื่อง |
| **Stack** | การนำ Template ไปใช้จริงหนึ่งครั้ง เช่น Stack ของกลุ่ม 01 |
| **Resource** | รายการที่ Stack จัดการ เช่น Port ของ Neutron หรือ Instance ของ Nova |
| **Dependency** | ความสัมพันธ์ที่กำหนดลำดับ เช่น VM ต้องรอให้ Port พร้อมก่อน |
| **Output** | ค่าที่ Template ส่งออก เช่น UUID ของ VM และ Port |

**Heat API** รับคำขอจัดการ Stack ส่วน **Heat Engine** อ่าน Template หาความสัมพันธ์ระหว่าง Resources แล้วเรียก API ของบริการต่าง ๆ พร้อมบันทึกสถานะและ Events ให้ตรวจย้อนหลัง ในตัวอย่างนี้ Heat สร้าง Port ผ่าน Neutron จากนั้นส่ง UUID ของ Port ให้ Nova ใช้ตอนสร้าง VM ส่วน Network, Image และ Flavor เป็นทรัพยากรเดิมที่ส่งเข้ามาเป็นพารามิเตอร์

## อ่านไฟล์ HOT ให้เป็น

| ส่วน | หน้าที่ |
|---|---|
| `heat_template_version` | ระบุรุ่นของไวยากรณ์ HOT รุ่นนี้เป็นคนละค่ากับ OpenStack Release เช่น Hibiscus (2026.2) |
| `parameters` | รับค่าเมื่อสร้างหรืออัปเดต Stack เช่น Image UUID |
| `resources` | กำหนดชนิดและคุณสมบัติของทรัพยากรที่ Heat จัดการ |
| `outputs` | ส่งข้อมูลที่ผู้ใช้ต้องการตรวจหลังสร้าง Stack |

`get_param` อ่านค่าจากพารามิเตอร์ และ `get_resource` อ้างถึง ID ของ Resource ใน Template การใช้ `get_resource` จาก VM ไปยัง Port ทำให้ Heat รู้ว่าต้องสร้าง Port ก่อน หากต้องการอ่านคุณสมบัติเฉพาะของ Resource ใช้ `get_attr` กับ Attribute ที่ชนิด Resource นั้นรองรับ

### การอัปเดตและการขยายระบบ

การอัปเดตบางคุณสมบัติ เช่น `metadata` ของ `OS::Nova::Server` ทำได้กับ VM เดิม ส่วนคุณสมบัติอื่นอาจทำให้ Heat สร้าง Resource มาทดแทน ให้ตรวจเอกสารของแต่ละ Property ก่อนอัปเดต Stack ที่มีข้อมูลสำคัญ การลบ Stack จะลบทรัพยากรที่ Stack จัดการตามนโยบายการลบของแต่ละ Resource

สำหรับ Auto Scaling, Heat สามารถจัดการกลุ่ม Instance และ Scaling Policy ได้ หากต้องการขยายตาม Metric ต้องมีแหล่ง Metric และระบบ Alarm ที่ตั้งค่าไว้ เช่น Gnocchi และ Aodh กิจกรรมหลักใช้การอัปเดต Stack แบบสั่งเอง จึงทำได้โดยไม่ต้องเตรียม Telemetry

## Workshop ท้ายบท — สร้าง อัปเดต และลบ VM ด้วย HOT

**ระบบ** DevStack OpenStack Hibiscus (2026.2) · **เวลา** 45–60 นาที · **รูปแบบ** กลุ่มละ 2–3 คน

### สิ่งที่ผู้สอนเตรียม

- ตรวจว่า Project ของนักศึกษาใช้ Heat ได้ โดย `openstack catalog list` แสดงบริการประเภท `orchestration` และ `openstack stack list` สำเร็จ
- หาก CLI ไม่รู้จักคำสั่ง `stack` ให้ตรวจปลั๊กอิน `python-heatclient` ใน Python Environment เดียวกับ OpenStack CLI หากไม่มีบริการ `orchestration` ให้ตรวจการเปิด Heat ตาม [Lab 1](../labs/lab01-devstack.md) และคู่มือ Heat สำหรับ DevStack
- เตรียม Image ที่บูตได้, Flavor ที่อยู่ในโควตา และ Network ที่มี IPv4 Subnet พร้อม IP ว่างสำหรับสร้าง Port ให้ผู้เรียนใช้ Project เดียวกันใน CLI และ Horizon

### ขั้นที่ 1 — เตรียม Template และพารามิเตอร์ (10–15 นาที)

บันทึกเป็นไฟล์ **`lec09-stack.yaml`**

```yaml
heat_template_version: '2021-04-16'
description: Create one Neutron port and one Nova instance.

parameters:
  image_id:
    type: string
  flavor_id:
    type: string
  network_id:
    type: string
  lab_stage:
    type: string
    default: initial

resources:
  lab_port:
    type: OS::Neutron::Port
    properties:
      network: { get_param: network_id }

  lab_vm:
    type: OS::Nova::Server
    properties:
      image: { get_param: image_id }
      flavor: { get_param: flavor_id }
      networks:
        - port: { get_resource: lab_port }
      metadata:
        lab_stage: { get_param: lab_stage }

outputs:
  vm_id:
    value: { get_resource: lab_vm }
  port_id:
    value: { get_resource: lab_port }
```

ใช้ `openstack image list`, `openstack flavor list` และ `openstack network list` เพื่อหา ID ตามที่ผู้สอนกำหนด แล้วแทนค่าใน Bash ใช้เทอร์มินัลเดิมตลอดกิจกรรม

```bash
STACK='lec09-teamXX'
IMAGE_ID='แทนด้วย Image UUID'
FLAVOR_ID='แทนด้วย Flavor ID'
NETWORK_ID='แทนด้วย Network UUID'
```

### ขั้นที่ 2 — สร้าง Stack แล้วจับคู่ Resource (15–20 นาที)

```bash
openstack stack create --wait -t lec09-stack.yaml \
  --parameter "image_id=$IMAGE_ID" \
  --parameter "flavor_id=$FLAVOR_ID" \
  --parameter "network_id=$NETWORK_ID" "$STACK"

openstack stack resource list "$STACK"
openstack stack output show "$STACK" vm_id
openstack stack output show "$STACK" port_id
```

รอให้เป็น `CREATE_COMPLETE` แล้วบันทึก UUID ของ VM และ Port จาก Outputs ตรวจด้วย `openstack port list --server VM_UUID` โดยแทน `VM_UUID` ด้วยค่าจริง หากขึ้น `CREATE_FAILED` ให้ใช้ `openstack stack event list "$STACK"` หา Resource และเหตุผลที่ล้มเหลวก่อนเริ่มขั้นถัดไป

### ขั้นที่ 3 — อัปเดต Metadata แล้วตรวจ VM เดิม (10–15 นาที)

```bash
openstack stack update --existing --wait \
  --parameter lab_stage=verified "$STACK"
openstack stack output show "$STACK" vm_id
```

`--existing` ใช้ Template และค่าก่อนหน้าเป็นฐาน โดยคำสั่งนี้ส่งค่าใหม่เฉพาะ `lab_stage` เมื่อ Stack เป็น `UPDATE_COMPLETE` ให้เทียบ UUID กับผลก่อนอัปเดต แล้วตรวจ `openstack server show VM_UUID -c id -c properties` ว่า `lab_stage` เปลี่ยนเป็น `verified` หากอัปเดตล้มเหลว ให้ตรวจ Events และบันทึกสถานะจริงก่อนดำเนินการต่อ

### งานส่งและคืนทรัพยากร (10 นาที)

- ส่ง Stack Status, VM UUID, Port UUID และค่า Metadata ก่อนกับหลัง Update ใช้ `openstack server show VM_UUID -c id -c properties` เก็บค่าก่อนอัปเดตด้วย
- อธิบายว่า **Heat รู้ได้อย่างไรว่าต้องสร้าง Port ก่อน VM** และ **หลักฐานใดแสดงว่า Update ใช้ VM เดิม**
- ตรวจชื่อ Stack ของกลุ่มก่อนใช้ `openstack stack delete --wait "$STACK"` จากนั้นตรวจว่า Stack, VM และ Port ที่สร้างหายจากรายการ ส่วน Network ที่รับผ่านพารามิเตอร์ยังอยู่

## สไลด์การบรรยาย

📥 [ดาวน์โหลดสไลด์การบรรยายที่ 9 (PDF)](../assets/slides/OpenStack_Heat_Architecture.pdf){ .md-button }

## แหล่งข้อมูลเพิ่มเติม

- [Heat 2026.2 — HOT Specification](https://docs.openstack.org/heat/2026.2/template_guide/hot_spec.html)
- [Heat 2026.2 — OpenStack Resource Types](https://docs.openstack.org/heat/2026.2/template_guide/openstack.html)
- [Heat Client 2026.2 — Stack Commands](https://docs.openstack.org/python-heatclient/2026.2/cli/stack.html)
- [Heat and DevStack](https://docs.openstack.org/heat/latest/getting_started/on_devstack.html)
