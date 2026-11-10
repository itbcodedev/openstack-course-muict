# การบรรยายที่ 4: Nova — บริการ Compute

## วัตถุประสงค์การเรียนรู้

- อธิบายบทบาทของ Nova API, Conductor, Scheduler, Compute และ Placement เมื่อสร้าง VM
- เลือก Flavor และแยกหน้าที่ของ Image, Network และ Key Pair
- ทดลองวงจรชีวิต VM และอธิบายผลต่อ Compute Quota จากข้อมูลจริง

## Nova ทำอะไร?

**Nova** คือบริการ Compute ของ OpenStack ใช้สร้างและจัดการวงจรชีวิตของ VM หรือ **Instance** เช่น เปิด หยุด เปลี่ยนขนาด และย้ายโฮสต์ ผู้ใช้ส่งคำขอผ่าน Horizon หรือ CLI ส่วน `nova-compute` สั่ง Hypervisor บน Compute Host ให้ทำงานจริง ตัวอย่างในบทนี้ใช้ libvirt/KVM

| ส่วนประกอบ | หน้าที่ |
|---|---|
| **Nova API** | รับคำขอ Compute ตรวจสิทธิ์ตาม Policy และส่งงานไปยังส่วนที่เกี่ยวข้อง |
| **Nova Conductor** | ประสานงานสร้างและเปลี่ยนขนาด VM และเป็นตัวกลางสำหรับการเข้าถึงฐานข้อมูลจาก Compute |
| **Nova Scheduler** | เลือก Compute Host ตามทรัพยากรและเงื่อนไขของคำขอ |
| **Placement** | บริการที่ติดตามทรัพยากรที่โฮสต์มีและที่ถูกจัดสรร เช่น vCPU และ RAM เพื่อใช้ประกอบการเลือกโฮสต์ |
| **Nova Compute** | ใช้ไดรเวอร์สั่ง Hypervisor ให้สร้างและจัดการ VM |
| **Console Proxy** | เชื่อม Console จากผู้ใช้ไปยัง VM เช่น ผ่าน noVNC ในระบบที่ตั้งค่าไว้ |

### เมื่อกด Launch Instance

1. **รับคำขอ** — Horizon หรือ CLI ส่งรายละเอียดไปยัง Nova API พร้อม Token จาก Keystone
2. **เลือกโฮสต์** — Conductor ประสานงานกับ Scheduler ซึ่งใช้ข้อมูลจาก Placement และพิจารณาเงื่อนไขอื่นของคำขอ
3. **เตรียมและบูต VM** — Nova ส่งงานไปยัง Compute ที่เลือกผ่าน RPC เช่น ผ่าน RabbitMQ ใน DevStack เมื่อบูตจาก Image ระบบใช้ Image จาก Glance ใช้พอร์ตที่ Neutron จัดการ และใช้ Cinder หากมี Volume ตามคำขอ จากนั้น Compute สั่ง Hypervisor ให้เริ่ม VM
4. **ตรวจผล** — ดูสถานะผ่าน Horizon หรือ CLI แล้วดู Console หรือ Log เพื่อยืนยันว่าระบบปฏิบัติการบูตถึงจุดที่ต้องการ

ลำดับนี้สรุปบทบาทหลัก รายละเอียดการเตรียม Image, Network และ Volume ขึ้นกับวิธีบูตและการตั้งค่าของระบบ ดูวิธีตามรอยจาก UUID ใน [Lecture 02](lecture02-architecture.md)

## สิ่งที่ต้องเลือกเมื่อสร้าง VM

| ตัวเลือก | คำถามที่ใช้เลือก | ตัวอย่าง |
|---|---|---|
| **Image** | VM จะบูตระบบปฏิบัติการอะไร? | CirrOS สำหรับทดลอง |
| **Flavor** | VM ต้องใช้ vCPU, RAM และพื้นที่ดิสก์เท่าไร? | เลือกขนาดเล็กที่ Image รองรับและอยู่ในโควตา |
| **Network** | VM ต้องเชื่อมกับเครือข่ายไหน? | เครือข่ายของ Project |
| **Key Pair** | จะใช้กุญแจใดเข้า SSH? | นำ Public Key ของผู้เรียนเข้า OpenStack และเลือกตอนสร้าง VM หาก Image รองรับการติดตั้งกุญแจ |
| **Security Group** | พอร์ตของ VM อนุญาตทราฟฟิกอะไร? | ใช้กฎที่ผู้สอนเตรียมไว้ โดย Neutron จัดการกฎเครือข่าย |

ใช้ `openstack flavor list` เพื่อดู Flavor ที่บัญชีเข้าถึงได้ และ `openstack flavor show FLAVOR_ID` เพื่อดู vCPU, RAM และ Root Disk ก่อนเลือก Flavor สำหรับการบูตจาก Image หากใช้ Volume เป็น Boot Source ขนาดของ Volume เป็นค่าที่ต้องตรวจแยกต่างหาก

นำเข้ากุญแจด้วย `openstack keypair create --public-key ~/.ssh/id_ed25519.pub my-key` เมื่อมีไฟล์ Public Key ดังกล่าว เก็บ Private Key ไว้ที่เครื่องผู้ใช้

## Compute Quota

Quota คือเพดานการใช้ทรัพยากรของ Project ส่วน **Usage** คือจำนวนที่ระบบนับว่าใช้อยู่ ค่าจริงขึ้นกับการตั้งค่าของแต่ละ Project

| Compute Quota | สิ่งที่จำกัด |
|---|---|
| `instances` | จำนวน Instance |
| `cores` | จำนวน vCPU รวม |
| `ram` | หน่วยความจำรวมที่นับตามโควตา |

Floating IP เป็นโควตาเครือข่ายของ Neutron ให้ดูแยกจาก Compute Quota ใช้ **Project → Compute → Overview** ใน Horizon เพื่อดู Usage เทียบกับ Limit หรือใช้ `openstack quota show --compute PROJECT_ID` เพื่อดู Limit เมื่อบัญชีมีสิทธิ์

## วงจรชีวิตของ VM

| การกระทำ | ผลที่ใช้ตรวจสอบ |
|---|---|
| สร้าง VM | มี Instance UUID ใหม่ เมื่อสำเร็จสถานะเปลี่ยนเป็น `ACTIVE` |
| Stop | เมื่อเสร็จสถานะเป็น `SHUTOFF` โดย Instance UUID เดิมยังอยู่ และโดยปกติทรัพยากร Compute ของ VM ยังถูกนับในโควตา |
| Start | VM เดิมกลับเป็น `ACTIVE` เมื่อเปิดสำเร็จ |
| Delete | เมื่อเสร็จ VM หายจากรายการ และตรวจการคืน Compute Quota จาก Usage ได้ |

### Resize และ Migration

| งาน | ผลที่ต้องการ | เงื่อนไขที่ต้องตรวจ |
|---|---|---|
| **Resize** | เปลี่ยน Flavor ของ VM | ตรวจโควตา ทรัพยากร และว่า DevStack รองรับ Resize บนโฮสต์เดียวหรือมีโฮสต์ปลายทางที่ใช้ได้ |
| **Cold Migration** | ย้าย VM ไปอีกโฮสต์โดยใช้ Flavor เดิม | VM หยุดระหว่างย้าย และต้องมีโฮสต์ปลายทางที่เหมาะสม |
| **Live Migration** | ย้าย VM ที่กำลังทำงานไปอีกโฮสต์ | ต้องมีโฮสต์ปลายทางและการตั้งค่าที่รองรับ เช่น CPU, เครือข่าย และ Storage ที่เข้ากันได้ ระบบที่ใช้ดิสก์ภายในเครื่องอาจใช้ Block Migration ตามที่รองรับ |

หากทดลอง Resize ให้รอ `VERIFY_RESIZE` แล้วเลือก **Confirm** เพื่อใช้ Flavor ใหม่ หรือ **Revert** เพื่อกลับไปใช้ Flavor เดิม ใน DevStack แบบโฮสต์เดียว ให้เก็บ Migration เป็นหัวข้อสาธิตเมื่อมี Compute Host เพิ่ม

## Workshop ท้ายบท — หยุด เปิด และลบ VM แล้ววัดโควตา

**เวลา** 45–60 นาที · **รูปแบบ** กลุ่มละ 2–3 คน

**ก่อนเริ่ม** ใช้ DevStack ที่ติดตั้งแล้ว ผู้สอนเตรียม Project, Image, Flavor และ Network พร้อมตั้งค่า CLI ให้ใช้ Project เดียวกับ Horizon หากหลายกลุ่มใช้ Project เดียวกัน ให้ทดลองทีละกลุ่มเพื่อให้เปรียบเทียบ Usage ได้

### ขั้นที่ 1 — เลือกขนาดและสร้าง VM (15–20 นาที)

1. เปิด Overview ใน Horizon แล้วจด **Usage และ Limit** ของ Instances, vCPU และ RAM
2. ใช้ `openstack flavor list` และ `openstack flavor show FLAVOR_ID` โดยแทน `FLAVOR_ID` ด้วยค่าจริง เลือก Flavor ที่มีโควตาเหลือพอและอธิบายเหตุผล
3. สร้าง VM ชื่อ `lec04-teamXX-vm01` ผ่าน Horizon ตาม [Workshop ใน Lecture 01](lecture01-introduction.md) จด UUID และบันทึก Usage เมื่อสถานะเป็น `ACTIVE`

### ขั้นที่ 2 — หยุดและเปิด VM เดิม (15–20 นาที)

แทนค่าด้วย UUID ของ VM ทดสอบของกลุ่ม

```bash
INSTANCE_ID='แทนด้วย Instance UUID'
openstack server stop "$INSTANCE_ID"
openstack server show "$INSTANCE_ID" -c id -c status -c flavor
```

เรียก `server show` ซ้ำจนเห็น `SHUTOFF` แล้วจด Usage จาก Horizon จากนั้นเปิด VM เดิม

```bash
openstack server start "$INSTANCE_ID"
openstack server show "$INSTANCE_ID" -c id -c status -c flavor
```

รอจนเห็น `ACTIVE` แล้วจด Usage อีกครั้ง ถ้าได้ `ERROR` ให้เก็บรายละเอียดและใช้แนวทางตรวจ Log จาก Lecture 02

### ขั้นที่ 3 — คืนทรัพยากรและสรุปผล (15–20 นาที)

ตรวจว่า UUID ตรงกับ VM ของกลุ่ม แล้วลบด้วย `openstack server delete "$INSTANCE_ID"` ตรวจว่า VM หายจากรายการและรีเฟรชหน้า Overview เพื่อจด Usage หลังลบ

| จุดวัด | สถานะ | Instance UUID | Instances ที่ใช้ | vCPU ที่ใช้ | RAM ที่ใช้ |
|---|---|---|---|---|---|
| ก่อนสร้าง | | | | | |
| หลังสร้าง | | | | | |
| หลัง Stop | | | | | |
| หลัง Start | | | | | |
| หลัง Delete | | | | | |

**งานส่ง** ส่งตารางพร้อมหลักฐานจาก CLI และ Horizon แล้วตอบว่า **เมื่อ VM หยุดอยู่ กลุ่มได้โควตาคืนเท่ากับตอนลบหรือไม่** ให้ใช้ตัวเลขที่สังเกตได้อธิบายผล

## สไลด์การบรรยาย

📥 [ดาวน์โหลดสไลด์การบรรยายที่ 4 (PDF)](../assets/slides/OpenStack_Nova_Architecture_Blueprint.pdf){ .md-button }

## แหล่งข้อมูลเพิ่มเติม

- [Nova Architecture](https://docs.openstack.org/nova/latest/admin/architecture.html)
- [Nova Quotas](https://docs.openstack.org/nova/latest/admin/quotas.html)
- [Nova Resize and Cold Migration](https://docs.openstack.org/nova/latest/contributor/resize-and-cold-migrate.html)
- [Nova Live Migration](https://docs.openstack.org/nova/latest/admin/live-migration-usage.html)
- [Lab 3 — OpenStack CLI](../labs/lab03-cli.md)
