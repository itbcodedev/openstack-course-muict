---
title: "การบรรยายที่ 10: Ceilometer / Gnocchi — การวัดการใช้ทรัพยากร"
---

# การบรรยายที่ 10: Ceilometer / Gnocchi — การวัดการใช้ทรัพยากร

## วัตถุประสงค์การเรียนรู้

- แยกหน้าที่ของ Ceilometer, Gnocchi และ Aodh
- อ่านหน่วยและช่วงเวลาของ Metric ก่อนนำตัวเลขไปตีความ
- เปรียบเทียบอัตราการเพิ่มของ CPU time ระหว่างช่วงพักกับช่วงสร้างโหลด

## Telemetry ตอบคำถามอะไร?

ใน [บท Nova](lecture04-nova.md) เราใช้ Flavor กำหนดจำนวน vCPU และใช้สถานะ `ACTIVE` ตรวจว่า Instance ทำงานอยู่ ถ้าต้องการตอบว่า **VM ใช้ CPU มากขึ้นตอนไหน** ต้องดูข้อมูลหลายจุดตามเวลา ซึ่งเป็นงานของระบบ Telemetry

| บริการ | หน้าที่ | ตัวอย่าง |
|---|---|---|
| **Ceilometer** | เก็บข้อมูลจาก Pollster หรือ Notification แล้วส่งผ่าน Pipeline ไปยังปลายทางที่ตั้งค่าไว้ | อ่าน CPU time สะสมของ VM จาก libvirt |
| **Gnocchi** | จัดเก็บและสืบค้น Metric ตามเวลา พร้อมเชื่อม Metric กับ Resource | ดูค่า CPU time ตามช่วงเวลาของ Instance หนึ่ง |
| **Aodh** | ประเมินเงื่อนไข Alarm และเรียก Action ที่กำหนดไว้ | แจ้งเตือนเมื่อข้อมูลที่ใช้ประเมินเกิน Threshold |

เมื่อใช้ Gnocchi เป็นปลายทาง Ceilometer เก็บค่าแล้วส่งไปให้ Gnocchi ประมวลผล ผู้ใช้จึงอาจต้องรอทั้งรอบเก็บข้อมูลและรอบประมวลผลก่อนเห็นจุดใหม่ Aodh เป็นบริการเสริมสำหรับประเมินเงื่อนไขและต้องตั้งค่าแยกต่างหาก

## อ่าน Metric ให้ถูกหน่วย

| คำศัพท์ | ตัวอย่าง |
|---|---|
| **Resource** | Instance ที่เราติดตาม ระบุด้วย Resource ID ใน Gnocchi |
| **Meter / Metric** | Ceilometer ใช้ชื่อ Meter เช่น `cpu` ส่วน Gnocchi เก็บข้อมูลของ Resource เป็น Metric ที่ค้นด้วยชื่อหรือ ID ได้ |
| **Sample / Measure** | ค่าที่วัดพร้อมเวลา เมื่ออ่านจาก Gnocchi อาจได้ค่าที่รวมไว้ตามช่วงเวลา |
| **Granularity** | ความกว้างของแต่ละช่วงเวลา เช่น 60 หรือ 300 วินาที |
| **Archive Policy** | กำหนด Granularity, ระยะเก็บข้อมูล และวิธีรวมค่า เช่น `mean` |

| Meter จาก Ceilometer 2026.2 | หน่วย | ความหมาย |
|---|---|---|
| `cpu` | ns | เวลาที่ใช้ CPU แบบสะสม |
| `vcpus` | vCPU | จำนวน vCPU ที่จัดสรรให้ VM |
| `memory` | MiB | RAM ที่จัดสรรให้ VM |
| `memory.usage` | MiB | RAM ที่ VM ใช้ตามข้อมูลที่ Hypervisor รายงาน ขึ้นกับการรองรับ |

ตัวอย่าง ถ้า `cpu` เพิ่มขึ้น **30,000,000,000 ns** ในเวลา **60 วินาที** บน VM ที่มี **1 vCPU** อัตราการใช้ CPU โดยประมาณในช่วงนั้นคือ **50%** คำนวณจาก `ΔCPU time (ns) ÷ (Δเวลา (s) × 1,000,000,000 × จำนวน vCPU) × 100` ถ้าใช้ค่าจากช่วงเวลาที่ Gnocchi รวมแบบ `mean` ผลเป็นการประมาณจากความชันของค่าเฉลี่ยแต่ละช่วง ให้ใช้ Granularity เดียวกันและเลือกช่วงที่อยู่ภายในสภาวะเดียวกัน

## Aodh และการแจ้งเตือน

Alarm ที่อิง Metric ต้องกำหนดหน่วย วิธีรวมค่า Granularity และจำนวนช่วงที่ใช้ประเมินให้ตรงกับข้อมูลจริง การตั้ง Threshold เป็น 80 บน Metric `cpu` โดยตรงจะเปรียบเทียบกับ **นาโนวินาทีสะสม** หากโจทย์คือเตือนเมื่อใช้ CPU เกิน 80% ต้องมี Metric หรือวิธีประเมินที่ให้ผลเป็นเปอร์เซ็นต์ก่อน

สถานะหลักคือ `ok` เมื่อเงื่อนไขเตือนไม่เป็นจริง, `alarm` เมื่อเงื่อนไขเป็นจริง และ `insufficient data` เมื่อข้อมูลที่ใช้ประเมินยังไม่พอ ส่วนการขยายระบบอัตโนมัติต้องกำหนด Action และเชื่อมกับ Scaling Policy เช่นใน [บท Heat](lecture09-heat.md)

## Workshop ท้ายบท — เปรียบเทียบ CPU time ตอนพักและตอนมีโหลด

**ระบบ** DevStack Hibiscus (2026.2) · **เวลา** 45–60 นาที เมื่อรอบเก็บข้อมูลเหมาะกับคาบ · **รูปแบบ** กลุ่มละ 2–3 คน

### สิ่งที่ผู้สอนเตรียม

- ตรวจว่า DevStack เปิด **Ceilometer Compute Polling และการส่งข้อมูลไป Gnocchi** ก่อนคาบ ขั้นตอนติดตั้งใน Lab ของคอร์สยังไม่ได้ระบุการเปิด Telemetry หากจะเพิ่มบริการ ให้ใช้สภาพแวดล้อมทดสอบแยกและอ้างอิงคู่มือ Ceilometer สำหรับ DevStack
- เตรียมบัญชี Project ทดสอบที่อ่าน Metric ได้ ตรวจ `openstack catalog list` ว่ามีบริการประเภท `metric` และทดสอบด้วย Gnocchi CLI จากแพ็กเกจ `gnocchiclient`
- เตรียม VM Linux ที่มี **1 vCPU** บน libvirt และเปิด Console ได้ ตรวจว่า `timeout` ใช้งานได้ใน Guest จดรอบ Polling, Granularity และเวลาที่ต้องรอให้ Measure ปรากฏ ผู้สอนควรทดสอบให้ได้อย่างน้อยสองช่วงข้อมูลต่อสภาวะก่อนใช้กิจกรรมสด
- หากไม่มีบริการ Metric หรือรอบวัดยาวเกินคาบ ให้ใช้ชุดข้อมูลจำลองท้ายกิจกรรม และระบุในรายงานว่าเป็นผลจากการคำนวณบนข้อมูลจำลอง

### ขั้นที่ 1 — หา Metric ของ Instance

ในเทอร์มินัลที่ใช้ Credentials ของ Project เดียวกับ VM ให้แทน Instance UUID ด้วยค่าจริง

```bash
INSTANCE_ID='แทนด้วย Instance UUID'
openstack server show "$INSTANCE_ID" -c id -c status -c flavor
gnocchi resource show --type instance "$INSTANCE_ID"
```

ตรวจว่า Resource ตรงกับ VM แล้วหา Metric `cpu` จากผลลัพธ์ หากค้นด้วย Instance UUID ไม่พบ ให้ใช้ `gnocchi resource list --type instance` และให้ผู้สอนช่วยยืนยัน Resource ID ที่ตรงกับ VM ก่อนดำเนินการต่อ

```bash
RESOURCE_ID='แทนด้วย Gnocchi Resource ID ที่ยืนยันแล้ว'
gnocchi measures show --resource-id "$RESOURCE_ID" --aggregation mean cpu
```

จด Timestamp, Granularity และค่า `cpu` เลือกใช้ Granularity เดียวตลอดการทดลอง หากไม่พบ Resource หรือ Measure ให้ตรวจสิทธิ์, การตั้งค่า Polling/Publisher และรอบประมวลผลของ Gnocchi ข้อมูลที่ยังไม่ปรากฏให้บันทึกเป็นข้อมูลไม่พอ

### ขั้นที่ 2 — บันทึกช่วงพักและสร้างโหลดใน Guest

1. ปล่อย VM ทำงานโดยไม่มีโหลดทดสอบ และบันทึกช่วงเวลาพักเป็น UTC ให้ครอบคลุมอย่างน้อยสองช่วงข้อมูลเต็มที่ใช้เปรียบเทียบได้
2. ใน **Console ของ VM ทดสอบ** ใช้ `date -u` บันทึกเวลาก่อนและหลังสร้างโหลด ตัวอย่างจำกัดโหลดหนึ่ง process ไว้ **180 วินาที** ใช้ค่านี้เมื่อผู้สอนยืนยันว่าครอบคลุมรอบวัดของ Lab สามารถกด Ctrl+C เพื่อหยุดก่อนครบเวลา

```bash
date -u
timeout 180 sh -c 'while :; do :; done'
date -u
```

3. รอให้ Gnocchi ประมวลผล แล้วเรียก `gnocchi measures show` ซ้ำ เลือกสองจุดที่ติดกันในช่วงพักและสองจุดที่ติดกันในช่วงโหลด โดยใช้ช่วงวัดที่อยู่ในสภาวะเดียวกันและ Granularity เท่ากัน หากได้จุดไม่พอให้บันทึกตามจริงหรือใช้ข้อมูลจำลอง หลีกเลี่ยงการนับช่วงเปลี่ยนสภาวะรวมกับช่วงเปรียบเทียบ

### ทางเลือกเมื่อ Telemetry ยังไม่พร้อม — ข้อมูลจำลอง

VM มี **1 vCPU** และใช้ Granularity **60 วินาที** ตัวเลขนี้สร้างขึ้นเพื่อฝึกคำนวณ

| สภาวะ | Timestamp UTC | CPU time สะสม (ns) |
|---|---|---:|
| พัก | 02:00:00 | 4,000,000,000 |
| พัก | 02:01:00 | 5,200,000,000 |
| โหลด | 02:04:00 | 23,200,000,000 |
| โหลด | 02:05:00 | 71,200,000,000 |

คำนวณจากผลต่าง **ภายในแต่ละสภาวะ** แล้วเปรียบเทียบอัตราที่ได้

### งานส่งและคืนทรัพยากร

- ส่งตาราง Timestamp, Granularity, ค่า `cpu`, ผลต่าง และเปอร์เซ็นต์โดยประมาณของแต่ละสภาวะ ระบุว่าเป็นข้อมูลจาก DevStack หรือข้อมูลจำลอง
- ตอบว่า **เหตุใดค่า CPU time สะสมจึงอาจเพิ่มทั้งตอนพักและตอนมีโหลด และควรเทียบค่าใดเพื่อบอกว่าโหลดสูงขึ้น?**
- หยุดโหลดเมื่อเสร็จและเก็บหลักฐานก่อนลบเฉพาะ VM ที่กลุ่มสร้างเพื่อ Workshop นี้ หากใช้ VM ที่ผู้สอนเตรียมไว้ ให้คืนตามที่กำหนด

## สไลด์การบรรยาย

📥 [ดาวน์โหลดสไลด์การบรรยายที่ 10 (PDF)](../assets/slides/OpenStack_Telemetry_Architecture.pdf){ .md-button }

## แหล่งข้อมูลเพิ่มเติม

- [Ceilometer 2026.2 — System Architecture](https://docs.openstack.org/ceilometer/2026.2/admin/telemetry-system-architecture.html)
- [Ceilometer 2026.2 — Measurements](https://docs.openstack.org/ceilometer/2026.2/admin/telemetry-measurements.html)
- [Ceilometer — DevStack Sandbox](https://docs.openstack.org/ceilometer/latest/contributor/devstack.html)
- [Gnocchi Documentation](https://gnocchi.osci.io/)
- [Aodh Documentation](https://docs.openstack.org/aodh/latest/)
