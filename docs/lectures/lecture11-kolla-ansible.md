# การบรรยายที่ 11: ติดตั้ง OpenStack ด้วย Kolla-Ansible

## วัตถุประสงค์การเรียนรู้

- อธิบายว่า Kolla-Ansible ติดตั้ง OpenStack อย่างไร และเลือกใช้ต่างจาก DevStack ในงานใด
- อ่าน Inventory, การตั้งค่าเครือข่าย และผลของ Prechecks
- ตรวจสอบ OpenStack Hibiscus (2026.2) แบบ All-in-One ตั้งแต่ Container จนถึง OpenStack API

## Kolla-Ansible ทำอะไร?

ในบทก่อนเราใช้ DevStack เพื่อเรียนรู้บริการของ OpenStack บทนี้เปลี่ยนมาดูงานของผู้ดูแลระบบว่า **จะติดตั้งบริการชุดเดียวกันซ้ำได้อย่างไร** โครงการ **Kolla** สร้าง Container Images ของบริการ ส่วน **Kolla-Ansible** ใช้ Ansible เพื่อเตรียมโฮสต์ สร้างไฟล์ตั้งค่า และรันบริการจาก Images เหล่านั้น

| คำถาม | DevStack | Kolla-Ansible |
|---|---|---|
| ใช้ทำอะไร? | พัฒนาและทดสอบ OpenStack รวมถึงงาน Lab | ติดตั้งและดูแล OpenStack ด้วย Container ทั้งแบบประเมินผลและแบบใช้งานจริง |
| กำหนดค่าที่ไหน? | `local.conf` และปลั๊กอิน DevStack | Inventory, `globals.yml` และไฟล์ตั้งค่าอื่นตามบริการ |
| ใช้กี่เครื่อง? | Lab ของคอร์สใช้เครื่องเดียว | เริ่มจาก All-in-One แล้วแยกบทบาทไปหลายโฮสต์ได้ |

## จาก All-in-One ไปสู่ Production

Inventory บอกว่าโฮสต์ใดทำหน้าที่อะไร ใน All-in-One โฮสต์เดียวทำหลายบทบาท เมื่อแยกหลายโฮสต์ ให้อ้างอิง Inventory `multinode` ของ Kolla-Ansible รุ่นที่ใช้

| บทบาท | สิ่งที่รับผิดชอบ |
|---|---|
| **Control** | API, Identity, Scheduler และบริการประกอบ เช่น MariaDB และ RabbitMQ ตามการตั้งค่า |
| **Compute** | รัน VM ผ่าน Nova Compute และ Hypervisor |
| **Network** | เชื่อมต่อเครือข่ายภายนอกตาม Backend และ Topology ที่เลือก เช่น OVN Gateway |
| **Storage** | จัดเก็บข้อมูลตาม Backend ที่เลือก ตัวอย่างเช่นใช้ Ceph ที่เตรียมและดูแลแยกจาก Kolla-Ansible |

ในระบบหลายโฮสต์ **HAProxy** กระจายคำขอ API และ **Keepalived** ดูแล Virtual IP (VIP) ตามการตั้งค่า เมื่อมีเพียงโฮสต์เดียว VIP ใช้เป็นที่อยู่สำหรับเข้าถึง API ได้ แต่ไม่มีโฮสต์สำรองเมื่อเครื่องเสีย ระบบ Production ยังต้องออกแบบ Quorum ของบริการที่เกี่ยวข้อง วางแผน TLS และทดสอบ Failover กับการกู้คืนจริง

### ไฟล์สำคัญ

| ไฟล์ | คำถามที่ใช้ตอบ |
|---|---|
| `all-in-one` หรือ `multinode` | Ansible จะรันงานบนโฮสต์ใดและให้แต่ละโฮสต์รับบทบาทอะไร? |
| `/etc/kolla/globals.yml` | ใช้ Release, Container Image, Interface และ VIP ใด? |
| `/etc/kolla/passwords.yml` | ใช้ข้อมูลลับอะไรกับบริการต่าง ๆ? |
| `/etc/kolla/clouds.yaml` | OpenStack CLI ใช้ Endpoint และ Credentials ใดหลังติดตั้ง? |

## Workshop ท้ายบท — ตรวจการติดตั้ง Kolla-Ansible Hibiscus

**เวลา** 60–90 นาทีสำหรับ Prechecks และตรวจผลบนระบบที่ผู้สอนเตรียม · **รูปแบบ** กลุ่มละ 2–3 คน การติดตั้งเต็มเป็นงานสาธิตหรือ Lab เพิ่มเติมตามเวลาและความเร็วในการดาวน์โหลด Images

### สิ่งที่ผู้สอนเตรียม

- ใช้ VM หรือเครื่อง **Ubuntu 24.04 LTS x86_64 แยกจาก DevStack** พร้อม Snapshot ก่อนติดตั้ง คู่มือระบุขั้นต่ำ 2 Network Interfaces, RAM 8 GB และพื้นที่ดิสก์ 40 GB ตรวจทรัพยากรเพิ่มหากจะบูต Instance และตรวจ `/dev/kvm` หากจะใช้ KVM ใน VM ซ้อน
- จัดเตรียม Management Interface ที่มี IP และใช้เข้าถึงโฮสต์ พร้อม VIP ที่ยังไม่มีผู้ใช้บนเครือข่ายเดียวกัน เตรียม External Interface ที่เปิดใช้งานแต่ไม่มี IP และใช้เฉพาะ Lab เพราะ Kolla จะนำไปผูกกับ Bridge ของ Neutron
- เตรียมสิทธิ์ `sudo` สำหรับการติดตั้ง และเตรียมเครื่อง All-in-One ที่ Deploy สำเร็จแล้วเป็นทางเลือกสำหรับช่วงตรวจผล เก็บ Lab ไว้บนเครือข่ายแยกหากใช้การตั้งค่า HTTP ตาม Quick Start

### ขั้นที่ 1 — อ่าน Inventory และวางแผนเครือข่าย

บนเครื่อง Lab ใหม่ รันตาม [Quick Start 2026.2](https://docs.openstack.org/kolla-ansible/2026.2/user/quickstart) ใช้ชุดคำสั่งนี้เพื่อเตรียม Kolla-Ansible รุ่นของ Hibiscus

```bash
sudo apt update
sudo apt install git python3-dev libffi-dev gcc libssl-dev \
  libdbus-glib-1-dev python3-venv
python3 -m venv ~/kolla-2026.2-venv
source ~/kolla-2026.2-venv/bin/activate
python -m pip install -U pip
python -m pip install 'git+https://opendev.org/openstack/kolla-ansible@stable/2026.2'
kolla-ansible install-deps

mkdir -p ~/kolla-lab
cd ~/kolla-lab
sudo mkdir -p /etc/kolla
sudo chown "$(id -un):$(id -gn)" /etc/kolla
cp -r "$VIRTUAL_ENV"/share/kolla-ansible/etc_examples/kolla/* /etc/kolla/
cp "$VIRTUAL_ENV"/share/kolla-ansible/ansible/inventory/all-in-one .
ansible-inventory -i ./all-in-one --graph
ip -br address
ip route
```

ดูว่า `localhost` อยู่ในกลุ่มใดบ้าง จากนั้นแก้ค่าต่อไปนี้ใน **`/etc/kolla/globals.yml`** โดยใช้ชื่อ Interface และ VIP ของเครื่องจริง หากมี Key อยู่แล้วให้แก้รายการเดิม

```yaml
kolla_base_distro: "ubuntu"
openstack_release: "2026.2"
network_interface: "REPLACE_WITH_MANAGEMENT_INTERFACE"
neutron_external_interface: "REPLACE_WITH_DEDICATED_EXTERNAL_INTERFACE"
kolla_internal_vip_address: "REPLACE_WITH_UNUSED_MANAGEMENT_IP"
```

จดว่า Interface ใดใช้เข้าถึงโฮสต์ Interface ใดให้ Neutron ใช้ และ IP ใดเป็น VIP ของ API ระวังอย่าเลือก Interface ที่ใช้ SSH เป็น External Interface

### ขั้นที่ 2 — Bootstrap และ Prechecks

```bash
kolla-genpwd
chmod 600 /etc/kolla/passwords.yml
kolla-ansible bootstrap-servers -i ./all-in-one
kolla-ansible prechecks -i ./all-in-one
```

Bootstrap เตรียมโฮสต์และ Container Engine ส่วน Prechecks ตรวจเงื่อนไขก่อนติดตั้ง หากพบข้อผิดพลาด ให้จดชื่อ Task และสาเหตุที่แสดง แล้วแก้ตามผลก่อนรัน Prechecks ซ้ำ เมื่อผ่านจึงเริ่ม Deploy หรือย้ายไปเครื่องที่ผู้สอนเตรียมไว้สำหรับขั้นถัดไป

### ขั้นที่ 3 — Deploy และตรวจผลจากสองมุม

บนเครื่องที่ผ่าน Prechecks แล้ว ผู้สอนสามารถสาธิตการติดตั้งและสร้างไฟล์ Credentials ด้วยคำสั่งต่อไปนี้ รัน Post-deploy เมื่อ Deploy สำเร็จ

```bash
kolla-ansible deploy -i ./all-in-one
kolla-ansible post-deploy -i ./all-in-one
python -m pip install python-openstackclient \
  -c https://releases.openstack.org/constraints/upper/2026.2
```

ในตัวอย่างที่ใช้ Docker ให้ตรวจ Container จากโฮสต์ แล้วทดสอบ API ผ่าน CLI ด้วยบัญชี Lab ที่ผู้สอนจัดให้ หากใช้ไฟล์จาก Post-deploy ให้ตรวจชื่อ Profile ใน `/etc/kolla/clouds.yaml` บนเครื่องและแทนค่าด้านล่าง เก็บไฟล์ Credentials ไว้ในเครื่องที่ได้รับอนุญาต

```bash
sudo docker ps --format 'table {{.Names}}\t{{.Status}}'

export OS_CLIENT_CONFIG_FILE=/etc/kolla/clouds.yaml
CLOUD='REPLACE_WITH_LAB_CLOUD_PROFILE'
openstack --os-cloud "$CLOUD" service list
openstack --os-cloud "$CLOUD" endpoint list --service identity
openstack --os-cloud "$CLOUD" compute service list
```

เทียบชื่อ Container กับ Service ที่ลงทะเบียนใน Keystone ตรวจว่า Endpoint ของ Identity ใช้ที่อยู่ใด และดูว่า `nova-compute` รายงาน `up` หรือไม่ หากผู้สอนเตรียม Image, Flavor, Network และ Project สำหรับผู้เรียนไว้แล้ว ให้สร้าง VM ทดสอบตาม [Workshop บทที่ 1](lecture01-introduction.md) เพื่อยืนยันถึงขั้นบูต Guest แล้วลบ VM ของกลุ่มหลังเก็บหลักฐาน

### งานส่ง

ส่งผล Prechecks หรือระบุว่าใช้เครื่องที่ Deploy ไว้แล้ว พร้อมตาราง **บทบาทโฮสต์ / Interface / VIP** และหลักฐานสองระดับคือสถานะ Container กับผลตอบกลับจาก OpenStack API อธิบายว่า **เหตุใด Container ที่แสดง Up จึงยังต้องตรวจ API และสถานะ Compute เพิ่ม**

## การอัปเกรด

ก่อนอัปเกรด ให้ตรวจเส้นทางที่รองรับจากรุ่นต้นทางไปปลายทาง อ่าน Release Notes สำรองฐานข้อมูลและไฟล์ตั้งค่า และทดสอบบนระบบแยก คู่มือของรุ่นปลายทางเป็นตัวกำหนดลำดับการอัปเกรด ซึ่งรวมถึงรุ่น Kolla-Ansible และการเปลี่ยนแปลงที่บริการต้องทำ การเปลี่ยนค่า `openstack_release` เพียงอย่างเดียวจึงไม่ครบกระบวนการ

## สไลด์การบรรยาย

📥 [ดาวน์โหลดสไลด์การบรรยายที่ 11 (PDF)](../assets/slides/การติดตั้งระดับ%20Production%20ด้วย%20Kolla-Ansible.pdf){ .md-button }

## แหล่งข้อมูลเพิ่มเติม

- [Kolla-Ansible 2026.2 — Quick Start](https://docs.openstack.org/kolla-ansible/2026.2/user/quickstart)
- [Kolla-Ansible 2026.2 — Support Matrix](https://docs.openstack.org/kolla-ansible/2026.2/user/support-matrix)
- [Kolla-Ansible 2026.2 — Release Notes](https://docs.openstack.org/releasenotes/kolla-ansible/2026.2.html)
