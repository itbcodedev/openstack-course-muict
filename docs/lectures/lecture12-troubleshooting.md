# การบรรยายที่ 12: การวินิจฉัยและแก้ปัญหา OpenStack

## วัตถุประสงค์การเรียนรู้

- แยกปัญหาการยืนยันตัวตน การสร้าง VM การบูต Guest และการเชื่อมต่อเครือข่าย
- เชื่อมหลักฐานจาก Horizon, CLI และ Service Logs ด้วยเวลา, Resource UUID หรือ Request ID
- แก้การตั้งค่าทีละจุดและยืนยันผลด้วยการทดสอบเดิม

## เริ่มจากอาการที่วัดได้

คำว่า “VM ใช้งานไม่ได้” อาจหมายถึงสร้าง VM ไม่สำเร็จ บูต Guest ไม่ขึ้น หรือเข้าแอปผ่านเครือข่ายไม่ได้ ให้จด **คำสั่งหรือการกระทำที่ล้มเหลว, Project, เวลาพร้อมเขตเวลา และ UUID** ของทรัพยากร แล้วเลือกจุดตรวจตามอาการ

| อาการ | หลักฐานที่ควรดู | สิ่งที่ต้องแยกให้ออก |
|---|---|---|
| CLI แจ้ง `401` หรือ `403` | Cloud profile, Project และข้อผิดพลาดของคำสั่ง | `401` ให้เริ่มจาก Authentication หรือ Token ส่วน `403` ให้ตรวจ Scope, Role และ Policy |
| สร้าง VM แล้วเป็น `ERROR` | Instance Details, Fault และ Nova Events ที่บัญชีมีสิทธิ์ดู | หากพบ `NoValidHost` ให้ตรวจ Flavor, Availability Zone, สถานะ Compute และข้อกำหนดที่ Scheduler ใช้ รวมถึงข้อมูลจาก Placement |
| VM เป็น `ACTIVE` แต่ Guest ไม่พร้อม | Console หรือ Console Log | Nova รายงานสถานะ Instance ส่วน Guest อาจยังบูตหรือเริ่มบริการไม่สำเร็จ |
| Horizon แสดง Fixed IP แต่ Guest ใช้ IP ไม่ได้ | Neutron Port, Subnet, Console และ `ip -4 addr` ใน Guest | IP ที่ Neutron จัดสรรกับ IP ที่ Guest ตั้งค่าได้เป็นหลักฐานคนละจุด |
| SSH หมดเวลา หรือ Ping ไม่ตอบ | ต้นทาง ปลายทาง Route, Port, Security Group และสถานะใน Guest | ตรวจเส้นทางและกฎก่อน ส่วน SSH ที่ตอบ `Permission denied` ให้เริ่มจากชื่อผู้ใช้และกุญแจ |
| Volume ไม่เป็น `available` หรือ Attach ไม่สำเร็จ | `openstack volume show`, Attachments และสถานะของ VM | แยกปัญหาสร้าง Volume จากปัญหา Attach แล้วให้ผู้ดูแลตรวจ Backend และ Availability Zone ตามอาการ |

ก่อนแก้ ให้บันทึกผลตั้งต้นและตั้งสมมติฐานหนึ่งข้อ หลังแก้ให้ใช้คำสั่งเดิมจากต้นทางเดิมไปยังปลายทางเดิมเพื่อเทียบผล หากต้องส่ง Log ให้คัดเฉพาะส่วนที่เกี่ยวข้องและปิดบัง Token, Credentials และข้อมูลลับ

## เครื่องมืออ่านหลักฐาน

### จากบัญชีผู้เรียน

ใช้ OpenStack CLI ใน Project ที่มี VM ทดสอบ แล้วแทน UUID ด้วยค่าจริง

```bash
INSTANCE_ID='REPLACE_WITH_INSTANCE_UUID'
openstack server show "$INSTANCE_ID"
openstack port list --server "$INSTANCE_ID"
openstack console log show --lines 50 "$INSTANCE_ID"
```

หากบัญชีและ Compute API รองรับ ให้ใช้ `openstack server event list "$INSTANCE_ID"` เพื่อหา Request ID ของการดำเนินการล่าสุด การดู Events หรือ Console Log ขึ้นกับ Policy และการตั้งค่าจริง ใช้หน้า Instance Details หรือ Console ใน Horizon หาก CLI แสดงข้อมูลไม่ครบ

### จากโฮสต์ DevStack สำหรับผู้สอน

ตรวจชื่อ systemd Unit จริงก่อนเลือก Log บันทึกเวลา UTC จากกิจกรรมและใช้ช่วงเดียวกันตอนค้นหา

```bash
systemctl list-units --all --type=service 'devstack@*'
UNIT='REPLACE_WITH_DISCOVERED_UNIT'
SINCE='REPLACE_WITH_UTC_TIMESTAMP'
sudo journalctl --unit "$UNIT" --since "$SINCE" --utc --no-pager
```

ตัวอย่างรูปแบบเวลาที่ใส่ใน `SINCE` คือ `2026-11-10 03:00:00 UTC` ใช้ UUID หรือ Request ID เชื่อม Log กับเหตุการณ์ เวลาใกล้กันช่วยจำกัดช่วงค้นหา ส่วนการยืนยันว่าเป็นคำขอเดียวกันต้องอาศัยรายละเอียดประกอบ

บน **ML2/OVN** ให้ตรวจสถานะพอร์ตและ OVN ตามการติดตั้งจริง เพราะ OVN จัดการ DHCP และ Layer 3 ของตนเอง ส่วน `qdhcp-*` และ `qrouter-*` เป็นจุดตรวจสำหรับการติดตั้งแบบใช้ Neutron DHCP/L3 Agents ที่เกี่ยวข้อง หากใช้ **Kolla-Ansible** ให้ตรวจ Container และไฟล์ Log ตามการตั้งค่าของระบบ เช่น `/var/log/kolla`

เมื่อหลักฐานชี้ไปที่ RabbitMQ หรือ MariaDB ให้ผู้ดูแลตรวจสถานะและ Log แบบอ่านอย่างเดียว การล้าง Queue, แก้ฐานข้อมูล หรือ Restart บริการส่วนกลางเป็นการเปลี่ยนระบบที่ต้องมีแผนและขอบเขตชัดเจน

## Workshop ท้ายบท — VM มี Fixed IP แต่ Ping ไม่ผ่าน

**ระบบ** DevStack Hibiscus (2026.2) · **เวลา** 50–60 นาที · **รูปแบบ** กลุ่มละ 2–3 คน

### สิ่งที่ผู้สอนเตรียม

- ใช้ VM-A และ VM-B ใน Project และ IPv4 Subnet เดียวกันตาม [Workshop บท Neutron](lecture05-neutron.md) ทั้งสอง VM ต้องเปิด Console และใช้ `ip` กับ `ping` ได้ ใช้ Fixed IP สำหรับการทดสอบ
- ให้ VM-A ส่ง IPv4 Egress ได้ ส่วน VM-B ใช้ Security Group เฉพาะกิจกรรมที่เป็น Stateful เตรียมกฎอนุญาต ICMP Echo Request จาก Fixed IP ของ A แบบ `/32`
- ผู้สอนทดสอบว่า Ping ผ่านก่อน จากนั้นนำกฎ ICMP ที่เตรียมไว้เฉพาะ Lab ออกและทดสอบว่า Ping ไม่ผ่าน ตรวจว่าพอร์ต B ไม่มี Security Group อื่นที่อนุญาต ICMP จาก A และเก็บค่ากฎเดิมไว้สำหรับคืนสภาพ

### ขั้นที่ 1 — บันทึกอาการและทำให้เกิดซ้ำ (10–15 นาที)

จด Project, UUID ของ VM ทั้งสอง และเวลาทดสอบเป็น UTC ใช้ Console ของ VM-B ตรวจว่า Guest มี Fixed IPv4 ตามที่คาด จาก Console ของ VM-A ให้แทน `FIXED_IP_B` ด้วยค่าจริง แล้วรัน `ping -c 4 FIXED_IP_B` บันทึกจำนวนแพ็กเก็ตที่ได้รับตอบกลับ

### ขั้นที่ 2 — แยกสาเหตุด้วยหลักฐาน (15–20 นาที)

จากเครื่องที่ใช้ OpenStack CLI ให้แทน UUID และ ID ด้วยค่าจริง โดยอ่าน Port ID จากคำสั่งที่สอง และอ่าน Security Group ID จากผลของ Port

```bash
VM_B='REPLACE_WITH_VM_B_UUID'
openstack server show "$VM_B" -c id -c status -c addresses
openstack port list --server "$VM_B"

PORT_B='REPLACE_WITH_PORT_B_ID'
openstack port show "$PORT_B" -c fixed_ips -c security_group_ids

GROUP_B='REPLACE_WITH_SECURITY_GROUP_ID'
openstack security group rule list "$GROUP_B"
```

หากพอร์ตผูกหลายกลุ่ม ให้ตรวจทุกกลุ่ม เขียนสมมติฐานก่อนแก้ เช่น **Guest มี Fixed IP แล้ว แต่ Security Group ของพอร์ต B ยังไม่อนุญาต ICMP จาก A** ระบุว่าหลักฐานใดสนับสนุนสมมติฐาน และเหตุใดจึงยังไม่เริ่มแก้ Router

### ขั้นที่ 3 — แก้เฉพาะจุดและทดสอบซ้ำ (15 นาที)

จาก **Horizon → Project → Network → Security Groups → Manage Rules** เพิ่มกฎ IPv4 Ingress สำหรับ ICMP Echo Request โดยใช้ Fixed IP ของ VM-A แบบ `/32` ใน Security Group เฉพาะกิจกรรมของ VM-B หรือใช้คำสั่งใน [Workshop บท Neutron](lecture05-neutron.md)

ทดสอบ Ping อีกครั้งจาก Console ของ VM-A ไปยัง Fixed IP เดิมของ VM-B จดจำนวนแพ็กเก็ตตอบกลับ หากยังไม่ผ่าน ให้ตรวจกลุ่มที่ผูกกับพอร์ตจริง, Egress ของ A และการตอบ ICMP ใน Guest แล้วบันทึกผลตามจริง

### งานส่งและคืนทรัพยากร (10 นาที)

| อาการและเวลา UTC | หลักฐาน | สมมติฐาน | สิ่งที่แก้ | ผลทดสอบซ้ำ |
|---|---|---|---|---|
| | | | | |

แนบหลักฐานพอร์ต กฎ และ Ping ก่อน–หลัง แล้วตอบว่า **สถานะ VM และ Fixed IP บอกอะไรได้บ้าง และหลักฐานใดบอกว่าปัญหาอยู่ที่ Security Group** หลังเก็บหลักฐาน ให้คืนกฎหรือคืน VM ตามสภาพที่ผู้สอนกำหนด หากกลุ่มสร้างทรัพยากรใหม่เองให้ลบตามขั้นตอนของบท Neutron

## สไลด์การบรรยาย

📥 [ดาวน์โหลดสไลด์การบรรยายที่ 12 (PDF)](../assets/slides/Troubleshoot_OpenStack.pdf){ .md-button }

## แหล่งข้อมูลเพิ่มเติม

- [DevStack — Using Systemd](https://docs.openstack.org/devstack/latest/systemd.html)
- [Nova 2026.2 — Compute Schedulers](https://docs.openstack.org/nova/2026.2/admin/scheduling.html)
- [Neutron 2026.2 — ML2 Plug-in](https://docs.openstack.org/neutron/2026.2/admin/config-ml2.html)
- [OpenStack CLI — Server Events](https://docs.openstack.org/python-openstackclient/stein/cli/command-objects/server-event.html)
- [Lecture 02 — ตามรอยการสร้าง VM](lecture02-architecture.md)
