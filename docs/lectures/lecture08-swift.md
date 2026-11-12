---
title: "การบรรยายที่ 8: Swift — บริการ Object Storage"
---

# การบรรยายที่ 8: Swift — บริการ Object Storage

## วัตถุประสงค์การเรียนรู้

- เลือกใช้ Block, File หรือ Object Storage ตามรูปแบบการเข้าถึงข้อมูล
- อธิบาย Account, Container, Object, Proxy และ Ring
- อัปโหลด ดาวน์โหลด และตรวจสอบข้อมูลกับ Metadata บน DevStack Hibiscus (2026.2)

## Swift ทำอะไร?

ใน [Lecture 07](lecture07-cinder.md) เราแนบดิสก์ Cinder เข้า VM เพื่อใช้ Filesystem ในบทนี้เราจะใช้ **Swift** เก็บไฟล์ผ่าน HTTP API โดยอ้างถึงชื่อ Object ภายใน Container แอปสามารถอัปโหลดหรือดาวน์โหลดได้โดยใช้สิทธิ์ของตน

| ประเภท | แอปเข้าถึงอย่างไร | ตัวอย่าง |
|---|---|---|
| **Block** | เข้าถึงอุปกรณ์แบบดิสก์ โดย Guest จัดการ Filesystem ได้ | ดิสก์ VM หรือดิสก์ฐานข้อมูล |
| **File** | ใช้ไฟล์และไดเรกทอรีบน Filesystem ที่แชร์ผ่านเครือข่าย | พื้นที่แชร์งานผ่าน NFS |
| **Object** | ส่งคำขอผ่าน API โดยระบุ Container และชื่อ Object | ไฟล์ภาพ รายงาน และข้อมูลเก็บถาวร |

### Account, Container และ Object

- **Account** คือขอบเขตระดับบนสุด ใน DevStack ที่ใช้ Keystone มักสัมพันธ์กับ Project
- **Container** คือกลุ่มของ Objects ภายใน Account
- **Object** คือข้อมูลพร้อม Metadata เช่น `reports/day-01.txt` ภายใน Container ชื่อ `lab-results` เครื่องหมาย `/` เป็นส่วนหนึ่งของชื่อที่เครื่องมืออาจแสดงเป็นโฟลเดอร์เสมือน

Swift มี **Swift API** ของตนเอง การรองรับคำขอแบบ S3 ขึ้นกับการเปิดใช้และตั้งค่า Middleware กิจกรรมนี้ใช้ Swift CLI กับ Swift API

## เบื้องหลังการอัปโหลด

1. **Proxy Server** รับคำขอ ตรวจสิทธิ์ผ่านกลไก Authentication/Authorization ที่ตั้งไว้ แล้วใช้ Ring หาตำแหน่งสำหรับคำขอ
2. **Ring** จับคู่ชื่อในขอบเขต Account, Container หรือ Object กับ Partition และอุปกรณ์จัดเก็บ มี Account Ring, Container Ring และ Object Ring ตาม Storage Policy
3. **Storage Servers** ทำงานตามประเภทข้อมูล Account Server ดูแลรายการ Containers, Container Server ดูแลรายการ Objects และ Object Server จัดเก็บเนื้อหากับ Metadata ของ Object

ระบบอาจใช้ **Replication** เพื่อเก็บหลายสำเนา หรือใช้ **Erasure Coding** เพื่อแบ่งข้อมูลเป็นชิ้นที่ใช้กู้คืนได้ตาม Storage Policy ความทนทานขึ้นกับการกระจายข้อมูลและการติดตั้งจริง งานเบื้องหลังช่วยซ่อมและกระจายการเปลี่ยนแปลง รายการใน Container จึงอาจอัปเดตช้ากว่าการเข้าถึง Object โดยตรงในบางกรณี ส่วนการลบตามเวลาที่ตั้งไว้เป็นหน้าที่ของ **Object Expirer**

DevStack แบบเครื่องเดียวเหมาะกับการเรียนรู้ API การอัปโหลดและดาวน์โหลดสำเร็จยืนยันการเข้าถึงข้อมูลใน Lab ได้ ส่วนการทนต่อโฮสต์เสียต้องทดสอบแยกบนระบบที่มีการกระจาย Storage จริง

## Workshop ท้ายบท — ตรวจข้อมูลและ Metadata หลังดาวน์โหลด

**ระบบ** DevStack OpenStack Hibiscus (2026.2) · **เวลา** 40–55 นาที · **รูปแบบ** กลุ่มละ 2–3 คน

### สิ่งที่ผู้สอนเตรียม

- เปิด Swift และเตรียม Swift CLI (`python-swiftclient`) พร้อม Credentials ของ Project ทดสอบ ตรวจว่า `openstack catalog list` มีบริการประเภท `object-store` และ `swift stat` สำเร็จด้วยบัญชีผู้เรียน
- หากไม่มี Endpoint ให้ตรวจว่า DevStack เปิด Swift ตาม [Lab 1](../labs/lab01-devstack.md) และคู่มือ DevStack หาก Endpoint มีแต่เข้าไม่ได้ ให้ตรวจ Project, สิทธิ์ และการเข้าถึง Proxy ก่อนเริ่ม Workshop

### ขั้นที่ 1 — สร้าง Container และอัปโหลด (15–20 นาที)

ใช้ Bash ในไดเรกทอรีทดลองที่ยังไม่มีไฟล์ชื่อนี้ แทน `XX` ด้วยเลขกลุ่ม

```bash
CONTAINER='lec08-teamXX-objects'
OBJECT='lec08-source.txt'

printf '%s\n' 'MUICT Swift workshop — team XX' > "$OBJECT"
sha256sum "$OBJECT"

swift post "$CONTAINER"
swift upload "$CONTAINER" "$OBJECT"
swift list "$CONTAINER"
```

### ขั้นที่ 2 — เพิ่ม Metadata แล้วดาวน์โหลด (15–20 นาที)

```bash
swift post -m 'lesson:lecture08' "$CONTAINER" "$OBJECT"
swift stat "$CONTAINER" "$OBJECT"
swift download -o lec08-downloaded.txt "$CONTAINER" "$OBJECT"
sha256sum "$OBJECT" lec08-downloaded.txt
```

บันทึกชื่อ Object, ขนาด และ Metadata ที่เห็นจาก `swift stat` แล้วเปรียบเทียบ SHA-256 ของทั้งสองไฟล์ ค่าเท่ากันเป็นหลักฐานว่าเนื้อหาที่ดาวน์โหลดตรงกับไฟล์ต้นฉบับ การเปลี่ยน Metadata จึงตรวจแยกจากการเปลี่ยนเนื้อหาได้ หากรายการ Object ยังไม่ปรากฏหลังอัปโหลด ให้ตรวจผลจาก `swift stat` ด้วยชื่อ Object โดยตรงและจดสิ่งที่พบ

### งานส่งและคืนทรัพยากร (10–15 นาที)

- ส่งชื่อ Container, ชื่อ Object, Metadata ที่เพิ่ม และค่า SHA-256 ทั้งสองค่า อธิบายว่า **ค่า Hash ที่ตรงกันยืนยันอะไร และใช้ยืนยันจำนวนสำเนาบน Storage Nodes ได้หรือไม่**
- ลบ Object ทดสอบด้วย `swift delete "$CONTAINER" "$OBJECT"` แล้วใช้ `swift list "$CONTAINER"` ตรวจว่าไม่มี Object เหลือ จากนั้นลบ Container ว่างของกลุ่มด้วย `openstack container delete "$CONTAINER"` ตรวจชื่อก่อนลบ

## สไลด์การบรรยาย

📥 [ดาวน์โหลดสไลด์การบรรยายที่ 8 (PDF)](../assets/slides/OpenStack_Swift_Architecture.pdf){ .md-button }

## แหล่งข้อมูลเพิ่มเติม

- [Swift 2026.2 Administrator Guide](https://docs.openstack.org/swift/2026.2/admin/index.html)
- [Swift Architectural Overview](https://docs.openstack.org/swift/latest/overview_architecture.html)
- [DevStack Configuration — Swift](https://docs.openstack.org/devstack/latest/configuration.html#swift)
- [Swift CLI](https://docs.openstack.org/python-swiftclient/latest/cli/index.html)
