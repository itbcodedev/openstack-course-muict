# การบรรยายที่ 5: Neutron — บริการเครือข่าย

## วัตถุประสงค์การเรียนรู้

- อธิบายว่า Network, Subnet, Port, Router และ Security Group เกี่ยวข้องกับ VM อย่างไร
- แยก Fixed IP ออกจาก Floating IP และอธิบายเส้นทางจาก VM ไปยังเครือข่ายภายนอก
- ทดลองเปลี่ยนกฎเครือข่ายบน DevStack **OpenStack Hibiscus (2026.2)** และใช้ผลก่อน–หลังระบุสาเหตุที่ VM ติดต่อกันได้หรือไม่ได้

## Neutron ทำอะไร?

ใน [Lecture 04](lecture04-nova.md) เราใช้ Nova สร้าง VM เมื่อเลือก Network ตอนกด Launch ระบบใช้ **Neutron** จัดการจุดเชื่อมต่อของ VM จัดสรร IP จาก Subnet ที่กำหนด และบังคับใช้กฎเครือข่ายที่ผูกกับพอร์ตนั้น

| คำศัพท์ | หน้าที่ | ตัวอย่างใน Lab |
|---|---|---|
| **Network** | เครือข่าย Layer 2 ที่พอร์ตเชื่อมต่อ | เครือข่ายของกลุ่มสำหรับ VM สองเครื่อง |
| **Subnet** | กำหนดช่วง IP, Gateway และตัวเลือก DHCP/DNS ของ Network | `10.55.20.0/24` และ Gateway `10.55.20.1` หากผู้สอนจัดสรรช่วงนี้ |
| **Port** | จุดเชื่อมต่อกับ Network มี MAC Address และอาจมี Fixed IP หนึ่งหรือหลายค่า | พอร์ตของ VM-A ที่ดูได้จาก `openstack port list --server` |
| **Router** | ส่งแพ็กเก็ตระหว่าง Subnets และเชื่อมไปยัง External Network เมื่อมี External Gateway | Router ของกลุ่มเชื่อม Subnet ของ VM กับ External Network ที่ผู้สอนเตรียม |
| **Security Group** | ชุดกฎ Ingress และ Egress ที่ใช้กับ Port | อนุญาตให้ VM-A ส่ง ICMP Echo Request มายัง VM-B |
| **Floating IP** | IP จาก External Network ที่ผูกกับ Fixed IP ของ Port เพื่อให้เข้าถึงผ่านเส้นทางภายนอกที่รองรับ | IP ที่ผู้สอนทดสอบจากเครื่องซึ่งเข้าถึงเครือข่ายภายนอกของ DevStack ได้ |

### แพ็กเก็ตเดินทางอย่างไร?

| ต้นทาง → ปลายทาง | เส้นทางที่ใช้ |
|---|---|
| VM-A → Fixed IP ของ VM-B ใน Subnet เดียวกัน | ส่งผ่านเครือข่าย Layer 2 ของ Project กฎที่พอร์ตของทั้งสอง VM ยังมีผล |
| VM → ปลายทางนอก Project Network ผ่าน Router | VM ส่งผ่าน Gateway ของ Subnet จากนั้น Router ส่งต่อไปยัง External Network และใช้ SNAT หากเปิดไว้ |
| เครื่องทดสอบ → Floating IP ของ VM | เครื่องทดสอบต้องมีเส้นทางถึง Floating IP จากนั้นระบบแปลปลายทางไปยัง Fixed IP ของพอร์ต และกฎ Ingress ต้องอนุญาตทราฟฟิกนั้น |

External Network ของ DevStack อาจอยู่บน `br-ex` ที่เข้าถึงได้จากบางเครื่องเท่านั้น คำว่า External ใน Horizon จึงบอกบทบาทของเครือข่ายใน OpenStack ส่วนการเข้าถึงจาก Windows host หรืออินเทอร์เน็ตขึ้นกับการเชื่อมต่อและเส้นทางจริงของ Lab

### Provider Network และ Self-Service Network

- **Provider Network** เชื่อมกับโครงสร้างเครือข่ายที่ผู้ดูแลกำหนด เช่น **Flat** ที่ไม่มี VLAN Tag บน segment นั้น หรือ **VLAN** ที่ใช้ Tag แบ่ง segment
- **Self-Service Network** ให้ Project สร้างเครือข่ายเองได้ตาม Policy ระบบ ML2/OVN มักใช้ **Geneve** สำหรับ Overlay ส่วน **VXLAN** พบได้ในรูปแบบติดตั้งที่เลือกใช้และรองรับ
- **External** เป็นคุณสมบัติที่ให้ใช้ Network เป็น External Gateway ของ Router ได้ ส่วน **Provider** อธิบายวิธีเชื่อม Network เข้ากับโครงสร้างเครือข่าย ทั้งสองคำจึงตอบคนละคำถาม

## Neutron เบื้องหลังใน Hibiscus (2026.2)

ตัวอย่างนี้อิง **ML2/OVN** ที่อธิบายไว้ใน [Lab 1 — DevStack](../labs/lab01-devstack.md) ผู้สอนควรตรวจ Backend ของเครื่องจริงก่อนสาธิต เพราะ Release เดียวกันเลือก Backend ได้ตามการติดตั้ง

1. **Neutron API และ ML2** รับคำขอสร้าง Network, Subnet หรือ Port โดย Type Driver ดูแลชนิดของ network segment และ Mechanism Driver นำการตั้งค่าไปใช้กับ Backend
2. **ML2/OVN** ส่งการตั้งค่าเชิงตรรกะไปยัง OVN Northbound DB จากนั้น `ovn-northd` แปลงเป็นข้อมูลที่ใช้ควบคุมเครือข่ายใน Southbound DB
3. **`ovn-controller`** บนโฮสต์อ่านข้อมูลจาก Southbound DB แล้วตั้งค่า Open vSwitch (OVS) เพื่อส่งต่อแพ็กเก็ตตามเครือข่ายและกฎที่กำหนด

ใน ML2/OVN การส่งต่อ Layer 3 และ DHCP ใช้กลไกของ OVN ส่วน Metadata มีส่วนประกอบของตนเอง ชื่อบริการจริงอาจต่างกันตามการติดตั้ง หาก Lab ใช้ **ML2/OVS** แบบที่มี Neutron Agents จะพบ OVS Agent, L3 Agent และ DHCP Agent ตามการตั้งค่าของระบบ

## Workshop ท้ายบท — ทำไม VM อยู่ Subnet เดียวกันแล้ว Ping ไม่ผ่าน?

**เวลา** 50–60 นาที และเพิ่ม 15 นาทีสำหรับกิจกรรมเสริม · **รูปแบบ** กลุ่มละ 2–3 คน · **ระบบ** DevStack OpenStack Hibiscus (2026.2)

### สิ่งที่ผู้สอนเตรียม

- Project ของแต่ละกลุ่ม พร้อมโควตาสำหรับ VM สองเครื่อง พอร์ต เครือข่าย Subnet และ Security Group สองชุด
- Image ขนาดเล็กที่ล็อกอินผ่าน Console และใช้ `ip` กับ `ping` ได้ เช่น CirrOS ที่ติดตั้งไว้ และแจ้งวิธีล็อกอินให้ผู้เรียน
- CIDR สำหรับแต่ละกลุ่มที่ไม่ทับกับเส้นทางของ Lab และ OpenStack CLI ที่ใช้ Project เดียวกับ Horizon
- หากจะทำส่วน Floating IP ให้เตรียม External Network ที่ Project ใช้ได้ โควตา Router/Floating IP และเครื่องทดสอบที่มีเส้นทางถึง Floating IP จริง

### ขั้นที่ 1 — สร้าง Network และตรวจพอร์ต (20 นาที)

1. ใน Horizon ไปที่ **Project → Network → Networks** สร้าง Network และ IPv4 Subnet ตาม CIDR ที่ผู้สอนให้ เปิด DHCP แล้วจด Network ID, Subnet ID และ Gateway
2. สร้าง Security Group แยกสำหรับ VM-A และ VM-B เช่น `lec05-teamXX-a` และ `lec05-teamXX-b` โดยแทน `XX` ด้วยเลขกลุ่ม ตรวจว่า A อนุญาต IPv4 Egress และ B ยังไม่มีกฎ Ingress ที่อนุญาต ICMP จาก A ตรวจด้วยว่ากลุ่มใช้กฎแบบ Stateful สำหรับการทดลองนี้
3. สร้าง VM-A และ VM-B บน Network เดียวกัน ใช้ Security Group ของแต่ละ VM ตามข้อก่อนหน้า ตรวจที่พอร์ตว่าไม่มี Group เพิ่มเติมที่อนุญาต ICMP จาก A ไป B อยู่แล้ว จากนั้นรอให้ทั้งสอง VM บูตและเปิด Console ได้
4. ใช้คำสั่งต่อไปนี้กับ UUID ของ VM แต่ละเครื่อง แล้วเปรียบเทียบ Fixed IP กับผล `ip -4 addr` ใน Console ของ VM

```bash
VM_ID='แทนด้วย Instance UUID ของ VM'
openstack port list --server "$VM_ID"
```

### ขั้นที่ 2 — เปลี่ยนกฎทีละตัวแล้ววัดผล (20–25 นาที)

1. จาก Console ของ VM-A รัน `ping -c 4 FIXED_IP_B` โดยแทน `FIXED_IP_B` ด้วย IPv4 ของ VM-B บันทึกจำนวนแพ็กเก็ตตอบกลับ
2. ใน Horizon เพิ่มกฎ **Ingress, IPv4, ICMP Echo Request** ให้ Security Group ของ VM-B โดยระบุ IPv4 ของ VM-A เป็น CIDR แบบ `/32` หากใช้ CLI ให้แทนค่าจริงก่อนรัน

```bash
A_CIDR='แทนด้วย IPv4 ของ VM-A ตามด้วย /32'
B_GROUP='แทนด้วย ID ของ Security Group ของ VM-B'

openstack security group rule create --ingress --ethertype IPv4 \
  --protocol icmp --icmp-type 8 --icmp-code 0 --remote-ip "$A_CIDR" "$B_GROUP"
```

3. ทดสอบ Ping ซ้ำจากต้นทางเดิมไปปลายทางเดิม หากสำเร็จ ให้บันทึกว่ากฎ Ingress เปลี่ยนอะไร และเหตุใดเส้นทางใน Subnet เดียวกันจึงใช้ได้โดยไม่ต้องสร้าง Router
4. ถ้าผลต่างจากที่คาด ให้ตรวจ Fixed IP ใน Guest, กลุ่มที่ผูกกับพอร์ตจริง, IPv4 Egress ของ A, IPv4 Ingress ของ B และการตอบ ICMP ภายใน Guest ทีละจุด

### ขั้นที่ 3 — Router และ Floating IP เมื่อ External Network พร้อม (กิจกรรมเสริม 15 นาที)

1. ตรวจ External Network ที่มีด้วย `openstack network list --external` แล้วใช้เครือข่ายที่ผู้สอนกำหนด สร้าง Router ของกลุ่ม ตั้ง External Gateway และเพิ่ม Interface ที่เชื่อมกับ Subnet ของกลุ่ม
2. จัดสรร Floating IP จาก External Network แล้วผูกกับพอร์ตของ VM-B จดคู่ **Floating IP ↔ Fixed IP** จากหน้า Floating IPs
3. เพิ่มกฎ ICMP Ingress สำหรับ IPv4 ต้นทางที่ระบบเห็นจากเครื่องทดสอบ ตามที่ผู้สอนแจ้ง แล้ว Ping Floating IP จากเครื่องนั้น เปรียบเทียบกับการ Ping Fixed IP จาก VM-A หากเข้าไม่ถึง ให้ตรวจเส้นทางของเครื่องทดสอบ, Router Gateway, การผูก Floating IP และ Security Group ตามลำดับ

### งานส่งและคืนทรัพยากร (10–15 นาที)

| การทดสอบ | ต้นทาง | IP ปลายทาง | กฎที่มีผล | จำนวนแพ็กเก็ตตอบกลับ |
|---|---|---|---|---|
| ก่อนเพิ่มกฎ | VM-A | Fixed IP ของ VM-B | | |
| หลังเพิ่มกฎ | VM-A | Fixed IP ของ VM-B | | |
| ทดสอบ Floating IP (ถ้าทำ) | เครื่องทดสอบที่ผู้สอนกำหนด | Floating IP ของ VM-B | | |

ส่งตารางและหลักฐานพอร์ตของ VM-B พร้อมตอบว่า **ถ้า VM สองเครื่องอยู่ Subnet เดียวกัน การเพิ่ม Router จะแก้ปัญหาที่ Security Group ปิด ICMP ได้หรือไม่ เพราะอะไร?** หากไม่ได้ทำส่วนเสริม ให้อธิบายว่าเครื่องทดสอบต้องมีเส้นทางถึงเครือข่ายใดก่อนทดสอบ Floating IP

หลังส่งงาน ให้ถอดและคืน Floating IP ที่กลุ่มจัดสรร ลบ VM ทั้งสองแล้วรอให้พอร์ตที่สร้างพร้อม VM ถูกลบ จากนั้นถอด Router Interface และ External Gateway แล้วลบ Router ของกลุ่มหากสร้างไว้ ปิดท้ายด้วยการลบ Subnet, Network และ Security Group ที่กลุ่มสร้าง

## สไลด์การบรรยาย

📥 [ดาวน์โหลดสไลด์การบรรยายที่ 5 (PDF)](../assets/slides/OpenStack_Neutron_Architecture.pdf){ .md-button }

## แหล่งข้อมูลเพิ่มเติม

- [Neutron 2026.2 — Networking Concepts](https://docs.openstack.org/neutron/2026.2/install/concepts.html)
- [Neutron 2026.2 — ML2 Plug-in](https://docs.openstack.org/neutron/2026.2/admin/config-ml2.html)
- [OpenStack CLI — Network v2 Commands](https://docs.openstack.org/python-openstackclient/latest/cli/command-objects/network/v2/index.html)
- [Lab 3 — OpenStack CLI](../labs/lab03-cli.md)
- [เอกสาร Neutron (PDF)](../assets/slides/doc-neutron.pdf)
