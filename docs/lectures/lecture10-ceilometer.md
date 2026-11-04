# การบรรยายที่ 10: Ceilometer / Gnocchi — การตรวจสอบและเก็บ Metrics

## วัตถุประสงค์การเรียนรู้

- เข้าใจบทบาทของ Ceilometer และ Gnocchi ในการเก็บ Metrics
- รู้จัก Meter, Sample, Resource และ Event
- สร้าง Alarms สำหรับ Monitoring และ Auto-scaling
- ใช้งาน Aodh สำหรับ Alarm Evaluation

---

## Ceilometer คืออะไร?

**Ceilometer** เป็นบริการ Telemetry ของ OpenStack:

- **Metering** — เก็บข้อมูลการใช้งานทรัพยากร (CPU, RAM, Disk, Network)
- **Monitoring** — ติดตามสถานะของ VMs และ Services
- **Billing** — ข้อมูลสำหรับคิดค่าบริการ (Chargeback/Showback)
- **Trigger** — ส่งข้อมูลไปยัง Alarms หรือ Auto-scaling

---

## สถาปัตยกรรม Telemetry

```
┌─────────────────────────────────────────┐
│         Ceilometer                      │
│  ┌──────────────┐  ┌────────────────┐ │
│  │   Polling    │  │   Notification │ │
│  │   Agents     │  │   Agents       │ │
│  │ (ดึง Metrics)│  │ (รับ Events)  │ │
│  └──────┬───────┘  └───────┬────────┘ │
│         │                   │          │
│  ┌──────▼───────────────────▼────────┐ │
│  │         Gnocchi (Time-Series DB)  │ │
│  │         เก็บ Metrics แบบ Indexed   │ │
│  └───────────────────────────────────┘ │
└─────────────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────┐
│         Aodh (Alarm Service)            │
│         ประเมิน Alarms                  │
└─────────────────────────────────────────┘
```

### ส่วนประกอบ

| ส่วนประกอบ | หน้าที่ |
|-----------|------|
| **ceilometer-compute** | Polling Agent บน Compute Node — เก็บ VM Metrics |
| **ceilometer-central** | Polling Agent บน Control Node — เก็บ Non-VM Metrics |
| **ceilometer-notification** | รับ Notifications จาก Message Queue |
| **Gnocchi** | Time-Series Database — เก็บ Metrics แบบประสิทธิภาพสูง |
| **Aodh** | Alarm Service — ประเมินเงื่อนไขและ Trigger Actions |

---

## Meter และ Sample

### Meter Types

| Meter | หน่วย | คำอธิบาย |
|-------|------|---------|
| `cpu_util` | % | เปอร์เซ็นต์การใช้ CPU |
| `memory.usage` | MB | หน่วยความจำที่ใช้ |
| `disk.read.bytes` | B | จำนวน Byte ที่อ่านจากดิสก์ |
| `network.incoming.bytes` | B | ปริมาณ Traffic เข้า |
| `network.outgoing.bytes` | B | ปริมาณ Traffic ออก |
| `instance` | instance | จำนวน VMs |
| `volume.size` | GB | ขนาด Volume |

### Sample
Sample = การบันทึกค่า Meter ณ เวลาหนึ่ง:

```json
{
  "counter_name": "cpu_util",
  "counter_type": "gauge",
  "counter_unit": "%",
  "counter_volume": 45.2,
  "resource_id": "instance-uuid",
  "timestamp": "2024-11-04T10:00:00Z"
}
```

---

## การใช้งาน Ceilometer CLI

```bash
# แสดงรายการ Meters ทั้งหมด
openstack metric list

# แสดง Meters ของ Instance หนึ่ง
openstack metric list --resource-id <instance-uuid>

# ดูค่า Metrics ย้อนหลัง
openstack metric show cpu_util --resource-id <instance-uuid>

# ดูสถิติ (Min, Max, Avg, Sum, Count)
openstack metric aggregation --meter cpu_util \
  --resource-id <instance-uuid> \
  --granularity 300 \
  --aggregation avg

# ดู Measures (Time-Series Data)
gnocchi measures show --resource-id <instance-uuid> cpu_util
```

---

## Gnocchi — Time-Series Database

**Gnocchi** แทนที่ MongoDB/Database แบบเดิม:

| คุณสมบัติ | คำอธิบาย |
|----------|---------|
| **Time-Series** | เก็บข้อมูลตามช่วงเวลา สืบค้นเร็ว |
| **Aggregation** | คำนวณสถิติล่วงหน้า (1 min, 5 min, 1 hour, 1 day) |
| **Carbonara** | โครงสร้างข้อมูลแบบ Compressed |
| **S3/Swift/Ceph** | เก็บข้อมูลบน Object Store |
| **Metricd** | Worker ที่ประมวลผล Metrics แบบ Async |

```bash
# แสดงรายการ Resources
openstack metric resource list

# แสดง Metrics ของ Resource
openstack metric resource show <resource-id>

# ดู Measures
openstack metric measures show --resource-id <resource-id> cpu_util

# สร้าง Archive Policy
openstack metric archive-policy create \
  --definition "granularity:0:00:01,points:86400" \
  --aggregation-methods mean,max \
  high-frequency
```

---

## Alarms ด้วย Aodh

**Aodh** ประเมิน Alarms จาก Metrics ใน Gnocchi:

### ประเภท Alarm

| ประเภท | คำอธิบาย |
|--------|---------|
| **Threshold** | เปรียบเทียบค่า Metrics กับ Threshold |
| **Event** | Trigger เมื่อเกิด Event ใด Event หนึ่ง |
| **Composite** | รวมหลายเงื่อนไขด้วย AND/OR |
| **Load Balancer Member** | Health Check สำหรับ LB |

### สร้าง Threshold Alarm

```bash
# สร้าง Alarm: CPU > 80% ต่อเนื่อง 2 ครั้ง
openstack alarm create \
  --name high-cpu-alarm \
  --type gnocchi_resources_threshold \
  --metric cpu_util \
  --threshold 80 \
  --comparison-operator gt \
  --evaluation-periods 2 \
  --aggregation-method mean \
  --granularity 300 \
  --resource-id <instance-uuid> \
  --resource-type instance \
  --alarm-action "http://webhook.example.com/alert"
```

### Alarm States

```
┌─────────┐    เงื่อนไขเป็นจริง     ┌─────────┐
│  ok     │ ──────────────────────→ │ alarm   │
│  (ปกติ) │ ←────────────────────── │ (เตือน) │
└─────────┘    เงื่อนไขเป็นเท็จ    └─────────┘
       │
       ↓ insufficient data
   ┌─────────┐
   │ unknown │
   └─────────┘
```

---

## การใช้งาน Alarm

```bash
# แสดงรายการ Alarms
openstack alarm list

# ดูรายละเอียด Alarm
openstack alarm show high-cpu-alarm

# อัปเดต Alarm
openstack alarm update --threshold 70 high-cpu-alarm

# ลบ Alarm
openstack alarm delete high-cpu-alarm

# ดูประวัติ Alarm (Alarm History)
openstack alarm-history show high-cpu-alarm
```

---

## สไลด์การบรรยาย

📥 [ดาวน์โหลดสไลด์การบรรยายที่ 10 (PDF)](../assets/slides/OpenStack_Telemetry_Architecture.pdf){ .md-button }

## แหล่งข้อมูลเพิ่มเติม

- [Ceilometer Documentation](https://docs.openstack.org/ceilometer/latest/)
- [Gnocchi Documentation](https://gnocchi.osci.io/)
- [Aodh Documentation](https://docs.openstack.org/aodh/latest/)
