---
title: "การบรรยายที่ 6: Glance — บริการ Image"
---

# การบรรยายที่ 6: Glance — บริการ Image

## วัตถุประสงค์การเรียนรู้

- อธิบายความสัมพันธ์ระหว่าง Image, Flavor, Instance และ Volume
- อ่านรูปแบบไฟล์ สถานะ และ Visibility ของ Image จาก Glance
- อัปโหลด Image แบบ Private บน DevStack Hibiscus (2026.2) แล้วพิสูจน์ว่า VM บูตจาก Image นั้นได้

## Glance ทำอะไร?

ใน [Lecture 04](lecture04-nova.md) เราเลือก Image ตอนกด Launch Instance บทนี้จะตามไปดูว่า **ไฟล์ที่ใช้บูตมาจากไหน** และเราจะรู้ได้อย่างไรว่า VM ใช้ Image ที่ตั้งใจเลือก

| สิ่งที่เลือกหรือสร้าง | หน้าที่ | บริการที่เกี่ยวข้อง |
|---|---|---|
| **Image** | ต้นแบบดิสก์ เช่น CirrOS หรือ Ubuntu Cloud Image | Glance |
| **Flavor** | ขนาด VM เช่น vCPU, RAM และขนาด Root Disk เมื่อใช้ดิสก์ที่ Nova จัดการ | Nova |
| **Instance** | VM ที่สร้างจากคำขอและมี Instance UUID ของตัวเอง | Nova |
| **Volume** | ดิสก์แบบ Block Storage ที่ใช้เป็น Boot Source หรือแนบเป็นดิสก์ข้อมูลได้ | Cinder |

**Glance API v2** ให้ผู้ใช้อัปโหลด ค้นหา และดึง Image ผู้ใช้เห็น Image UUID และข้อมูลกำกับ เช่น ชื่อ รูปแบบไฟล์ สถานะ และ Visibility ส่วนไฟล์ Image อยู่ใน Backend Store ที่ผู้ดูแลกำหนด เช่น File Store หรือ Object Storage การบูตจาก Image ใช้ Image เป็นต้นแบบของดิสก์ VM การเขียนไฟล์ใน VM จะเปลี่ยนดิสก์ของ VM เครื่องนั้น

## อ่านข้อมูล Image ก่อนนำไปใช้

| ข้อมูล | ใช้ตอบคำถามอะไร |
|---|---|
| **ID** | เรากำลังเลือก Image รายการใด แม้ชื่อจะซ้ำกัน? |
| **Disk Format** | ข้อมูลดิสก์อยู่ในรูปแบบอะไร เช่น `qcow2` หรือ `raw`? |
| **Container Format** | ไฟล์มีโครงห่อหุ้มเพิ่มเติมหรือไม่? ค่า `bare` หมายถึงไม่มีโครงห่อหุ้ม |
| **Status** | ข้อมูลพร้อมให้บริการดึงไปใช้หรือยัง? |
| **Visibility** | Project ใดมีสิทธิ์เข้าถึง Image ตาม Policy? |
| **Min Disk / Min RAM** | ผู้เผยแพร่ Image ระบุข้อกำหนดขั้นต่ำไว้เท่าไร? |

### รูปแบบไฟล์และความเข้ากันได้

- **QCOW2** เป็นรูปแบบของ QEMU ที่รองรับการจัดสรรพื้นที่ตามการใช้งานและกลไก Copy-on-Write
- **RAW** เป็นข้อมูลดิสก์แบบตรงไปตรงมา ไฟล์ RAW อาจเป็น sparse file ได้ตามวิธีสร้างและระบบจัดเก็บ
- นามสกุลไฟล์อย่าง `.img` ใช้บอก Disk Format เพียงอย่างเดียวไม่ได้ ให้ดูข้อมูลจากผู้เผยแพร่หรือใช้ `qemu-img info` กับไฟล์จากแหล่งที่เชื่อถือได้
- ความสามารถในการบูตยังขึ้นกับสถาปัตยกรรม CPU, Firmware, รูปแบบดิสก์ และการรองรับของ Compute Host ข้อมูลกำกับ เช่น `hw_disk_bus` อาจมีผลต่อวิธีที่ Nova สร้าง VM จึงควรตั้งเมื่อทราบข้อกำหนดของ Image

### สถานะและการมองเห็น

การอัปโหลดไฟล์โดยตรงอาจผ่าน `queued` (สร้างรายการแล้ว) และ `saving` (กำลังรับข้อมูล) ก่อนเป็น `active` (ข้อมูลพร้อมให้ดึงไปใช้) การอัปโหลดไฟล์ขนาดเล็กอาจผ่านสถานะชั่วคราวเร็วมาก ใช้สถานะ `active` เป็นจุดตรวจฝั่ง Glance แล้วตรวจการบูตจาก Console ฝั่ง Nova แยกกัน

| Visibility | ขอบเขตการเข้าถึงตามปกติ |
|---|---|
| **Private** | Project เจ้าของ ภายใต้ Policy ของระบบ |
| **Shared** | Project เจ้าของและ Project ที่ได้รับการแชร์ การตอบรับมีผลต่อการแสดงในรายการ Image ตามปกติของผู้รับ |
| **Community** | Project อื่นเข้าถึงได้ แต่รายการทั่วไปของผู้ใช้รายอื่นจะไม่แสดงโดยอัตโนมัติ |
| **Public** | เปิดให้ผู้ใช้ใน Cloud ใช้ได้ตาม Policy การเผยแพร่มักสงวนไว้ให้ผู้ดูแล |

### การเตรียม Image สำหรับใช้ซ้ำ

เริ่มจาก Cloud Image ที่ผู้เผยแพร่เตรียมไว้ หรือสร้าง Image มาตรฐานด้วยเครื่องมืออย่าง Disk Image Builder ในกรณีที่สร้าง Snapshot จาก VM ให้ตรวจประเภท Boot Source และความสอดคล้องของข้อมูลก่อนนำไปใช้ซ้ำ พร้อมเอาข้อมูลเฉพาะเครื่องและข้อมูลลับออกจากต้นแบบ การเปรียบเทียบ SHA-256 ควรใช้ค่าที่ได้จากผู้เผยแพร่หรือผู้สอนผ่านช่องทางที่เชื่อถือได้

## Workshop ท้ายบท — อัปโหลด Private Image แล้วพิสูจน์ว่า VM บูตได้

**ระบบ** DevStack OpenStack Hibiscus (2026.2) · **เวลา** 45–60 นาที · **รูปแบบ** กลุ่มละ 2–3 คน

### สิ่งที่ผู้สอนเตรียม

- ไฟล์ Cloud Image ขนาดเล็กที่ทดสอบแล้วว่าเข้ากับ Compute Host เช่น CirrOS แบบ QCOW2 สำหรับโฮสต์ x86_64 พร้อมค่า SHA-256 ที่ตรวจสอบแล้ว
- Project ที่อัปโหลด Private Image และสร้าง VM ได้ พร้อม Flavor, Network และ CLI ที่ชี้ไป Project เดียวกับ Horizon
- วิธีเปิด Console และชื่อผู้ใช้สำหรับ Image หากต้องการล็อกอิน การทดลองใช้ Console เป็นหลัก

### ขั้นที่ 1 — ตรวจไฟล์ต้นฉบับ (10 นาที)

แทนชื่อไฟล์ด้วยไฟล์ที่ผู้สอนแจก แล้วเปรียบเทียบค่า SHA-256 กับค่าที่ได้รับก่อนอัปโหลด หากไม่ตรงให้หยุดและตรวจไฟล์กับผู้สอน หากมี `qemu-img` ให้ตรวจว่า `file format` เป็น `qcow2` สำหรับคำสั่งในขั้นถัดไป

```bash
IMAGE_FILE='./cirros-lab.qcow2'
sha256sum "$IMAGE_FILE"
qemu-img info "$IMAGE_FILE"
```

### ขั้นที่ 2 — อัปโหลดแล้วตรวจใน Glance (10–15 นาที)

แทน `XX` ด้วยเลขกลุ่ม และใช้ Bash session เดิมจากขั้นก่อนหน้า

```bash
IMAGE_NAME='lec06-teamXX-image'

openstack image create --private --disk-format qcow2 \
  --container-format bare --file "$IMAGE_FILE" "$IMAGE_NAME"
```

คัดลอก ID จากผลลัพธ์ แล้วตรวจข้อมูลด้วย ID

```bash
IMAGE_ID='แทนด้วย Image UUID'
openstack image show "$IMAGE_ID" \
  -c id -c name -c status -c visibility -c disk_format -c container_format
```

รอจน Image เป็น `active` และตรวจว่า Visibility เป็น `private` หากรายการยังเป็น `queued` หรือคำสั่งรายงานข้อผิดพลาด ให้บันทึกผลและตรวจไฟล์ สิทธิ์อัปโหลด และข้อจำกัดของ Project กับผู้สอนก่อนลองใหม่

### ขั้นที่ 3 — สร้าง VM และยืนยัน Boot Source (15–20 นาที)

1. ใน Horizon เปิด **Project → Compute → Instances → Launch Instance** ตั้งชื่อ `lec06-teamXX-vm01` แล้วเลือก **Boot Source: Image** และ Image ที่กลุ่มอัปโหลด หากมีตัวเลือกสร้าง Boot Volume ให้เลือก **No** สำหรับกิจกรรมนี้
2. เลือก Flavor และ Network ที่ผู้สอนเตรียม แล้ว Launch เมื่อ VM เป็น `ACTIVE` ให้เปิด Console เพื่อตรวจหลักฐานว่าระบบปฏิบัติการบูต หาก VM เป็น `ERROR` ให้เก็บข้อความจากหน้า Instance และตรวจความเข้ากันได้ของ Image กับ Flavor และโฮสต์
3. คัดลอก Instance UUID จาก Horizon ใช้คำสั่งด้านล่าง แล้วเทียบข้อมูลในช่อง `image` กับ Image UUID ที่ได้จาก Glance

```bash
INSTANCE_ID='แทนด้วย Instance UUID'
openstack server show "$INSTANCE_ID" -c id -c image -c status
```

### งานส่งและคืนทรัพยากร (10–15 นาที)

- ส่งค่า SHA-256 ที่ตรวจแล้ว ข้อมูล Image จาก Glance และข้อมูล Instance ที่เชื่อมกลับไปยัง Image UUID เดียวกัน พร้อมหลักฐานการบูตจาก Console
- ตอบว่า **ถ้า Glance แสดง Image เป็น `active` แต่ VM บูตไม่สำเร็จ เราทราบอะไรแล้ว และต้องตรวจอะไรต่อ?** แยกหลักฐานฝั่ง Glance ออกจากหลักฐานฝั่ง Nova
- ลบ VM ทดสอบของกลุ่มและตรวจว่าหายจากรายการก่อนลบ Private Image ที่กลุ่มอัปโหลด ตรวจ Image UUID ก่อนลบ

## สไลด์การบรรยาย

📥 [ดาวน์โหลดสไลด์การบรรยายที่ 6 (PDF)](../assets/slides/OpenStack_Glance_Architecture.pdf){ .md-button }

## แหล่งข้อมูลเพิ่มเติม

- [Glance 2026.2 Documentation](https://docs.openstack.org/glance/2026.2/)
- [OpenStack CLI 2026.2 — Image v2 Commands](https://docs.openstack.org/python-openstackclient/2026.2/cli/command-objects/image/v2/index.html)
- [Disk Image Builder](https://docs.openstack.org/diskimage-builder/latest/)
