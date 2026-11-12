---
title: "แทนที่การกำหนดค่าเดิมที่ชี้ไปที่ keystone-wsgi-public"
---

install 

cd /opt/stack

sudo apt install -y python3.11 python3.11-dev python3.11-venv
sudo pt install python3-virtualenv
virtualenv -p /usr/bin/python3.11  requirements/.venv/

cd devstack
./stack.sh


GLOBAL_VENV=False

https://medium.com/@ion.stefanache0/installing-openstack-using-devstack-under-virgin-fresh-ubuntu-24-04-lts-e3790280359b


sudo journalctl -u devstack@keystone

## manual install
cd /opt/stack/keystone && /opt/stack/data/venv/bin/pip install -e .

/opt/stack/data/venv/bin/pip install uwsgi


 /opt/stack/data/venv/bin/pip --list
## Environment
ความแตกต่างหลักระหว่างทั้งสองอยู่ที่วัตถุประสงค์และตำแหน่งการใช้งาน: /opt/stack/data/venv คือ Python environment แบบรวมศูนย์ที่ DevStack สร้างขึ้นเพื่อให้บริการ OpenStack ต่างๆ (เช่น Keystone) ทำงาน ในขณะที่ requirements/.venv คือ environment ชั่วคราวที่ DevStack ใช้เพื่อรันเครื่องมือจัดการ dependency ในระหว่างกระบวนการติดตั้งเอง


ความแตกต่างหลักระหว่างทั้งสองอยู่ที่**วัตถุประสงค์และตำแหน่งการใช้งาน**: `/opt/stack/data/venv` คือ Python environment แบบรวมศูนย์ที่ DevStack สร้างขึ้นเพื่อให้บริการ OpenStack ต่างๆ (เช่น Keystone) ทำงาน ในขณะที่ `requirements/.venv` คือ environment ชั่วคราวที่ DevStack ใช้เพื่อรันเครื่องมือจัดการ dependency ในระหว่างกระบวนการติดตั้งเอง

### `/opt/stack/data/venv` (environment แบบรวมศูนย์ของบริการ)

นี่คือ environment ที่คุณกำลังตรวจสอบอยู่ และเป็นหนึ่งใน environment หลักที่ DevStack ใช้หลังจากติดตั้งเสร็จ

*   **วัตถุประสงค์**: ใช้สำหรับ**เรียกใช้โค้ดบริการ OpenStack** บริการอย่าง Keystone, Nova และอื่นๆ จะถูกติดตั้งในรูปแบบ editable ภายใน environment นี้ จากนั้นจึงเริ่มต้นผ่าน systemd 
*   **ความเชื่อมโยงกับปัญหาของคุณ**: ข้อผิดพลาด `ModuleNotFoundError: No module named 'keystone'` ที่คุณพบก่อนหน้านี้ หมายความว่าเมื่อ uWSGI เริ่มต้นบริการ Keystone มันไม่สามารถค้นหาแพ็กเกจ `keystone` ใน environment นี้ได้
*   **ตำแหน่ง**: `/opt/stack/data/venv` อยู่ภายใต้ home directory `/opt/stack` ของผู้ใช้ stack โดยเป็นสถานที่จัดเก็บ dependency แบบรวมศูนย์สำหรับบริการ OpenStack ทั้งหมด

### `requirements/.venv` (environment ของเครื่องมือติดตั้ง)

environment นี้โดยปกติจะอยู่ที่ `/opt/stack/requirements/.venv` และมีบทบาทในขั้นตอนการติดตั้ง

*   **วัตถุประสงค์**: ใช้สำหรับ**เรียกใช้เครื่องมือจัดการ dependency ของ OpenStack** ตัวอย่างเช่น สคริปต์ DevStack จะเรียกใช้ pip ใน environment นี้ พร้อมกับใช้ไฟล์ `upper-constraints.txt` เพื่อควบคุมและติดตั้ง dependency เวอร์ชันที่ถูกต้องสำหรับโครงการ OpenStack ต่างๆ 
*   **ความเชื่อมโยงกับปัญหาของคุณ**: หาก environment นี้มีปัญหา (เช่น pip หายไป) การติดตั้งมักจะล้มเหลวตั้งแต่เนิ่นๆ ด้วยข้อผิดพลาดเช่น `"/opt/stack/requirements/.venv/bin/pip" not exists` 
*   **สถานะ**: เป็นเหมือน "เครื่องมือ" ของกระบวนการติดตั้ง มากกว่าที่จะเป็น "ที่อยู่อาศัย" ของบริการที่กำลังทำงานอยู่

### สรุป

พูดง่ายๆ คือ:
*   **`requirements/.venv`** ใช้เพื่อ**ติดตั้ง** OpenStack
*   **`/opt/stack/data/venv`** ใช้เพื่อ**เรียกใช้** OpenStack

ปัญหาที่คุณกำลังเผชิญอยู่คือ บริการ Keystone ใน environment แรกไม่สามารถเริ่มต้นได้อย่างถูกต้อง

---
### เพื่อช่วยวิเคราะห์ปัญหาของคุณต่อไป
คุณช่วยรันคำสั่งต่อไปนี้ใน terminal แล้วให้ผลลัพธ์แก่ฉันได้ไหม เพื่อยืนยันว่า uWSGI ใน environment แบบรวมศูนย์สามารถใช้ได้จริงหรือไม่:

```bash
/opt/stack/data/venv/bin/python -c "import uwsgi; print('uwsgi module ok')"
```

หากคำสั่งนี้ล้มเหลว แสดงว่าปัญหาอยู่ที่ uWSGI ที่ติดตั้งใน venv ไม่สมบูรณ์


/opt/stack/data/venv/bin/pip  install --force-reinstall uwsgi


stack@openstack:~/devstack$ /opt/stack/data/venv/bin/pip  install --force-reinstall uwsgi
Collecting uwsgi
  Using cached uwsgi-2.0.31-cp311-cp311-linux_x86_64.whl
Installing collected packages: uwsgi
  Attempting uninstall: uwsgi
    Found existing installation: uWSGI 2.0.31
    Uninstalling uWSGI-2.0.31:
      Successfully uninstalled uWSGI-2.0.31
Successfully installed uwsgi-2.0.31
stack@openstack:~/devstack$ /opt/stack/data/venv/bin/python -c "import uwsgi; print('uwsgi module ok')"
Traceback (most recent call last):
  File "<string>", line 1, in <module>
ModuleNotFoundError: No module named 'uwsgi'
stack@openstack:~/devstack$


อ้า นี่คือจุดสำคัญที่ผมเข้าใจผิดไปก่อนหน้านี้ — venv ของคุณใช้ Python 3.11.18 ไม่ใช่ 3.10 แต่บันทึก Keystone ก่อนหน้านี้แสดง Python version: 3.10.12 ซึ่งหมายความว่า uWSGI ที่ systemd เรียกใช้เป็นตัวของระบบ (Python 3.10) ไม่ใช่ตัวใน venv (Python 3.11)

นี่คือสาเหตุที่แท้จริงของปัญหา ModuleNotFoundError: No module named 'keystone':

องค์ประกอบ	Python เวอร์ชัน	ตำแหน่ง
venv ที่ติดตั้ง keystone	3.11.18	/opt/stack/data/venv/bin/python
uWSGI ที่ systemd รัน	3.10.12	/usr/bin/uwsgi-core (ของระบบ)
uWSGI ของระบบใช้ Python 3.10 จึงมองไม่เห็นแพ็กเกจ keystone ที่ติดตั้งอยู่ใน venv Python 3.11


ยืนยันแล้ว: pip 26.3 คือสาเหตุที่ทำให้เกิดปัญหา คุณกำลังประสบกับข้อบกพร่องที่ทราบแล้วในฟอรัม OpenStack——pip เวอร์ชัน 25.3 ขึ้นไปได้ลบการสนับสนุนสำหรับการติดตั้งแบบ editable ในรูปแบบเก่า (setup.py develop) ส่งผลให้สคริปต์ keystone-wsgi-public ไม่ถูกสร้างขึ้น และ uWSGI จึงไม่สามารถโหลด Keystone ได้

ขั้นตอนแรก: ลดเวอร์ชัน pip
/opt/stack/data/venv/bin/pip install "pip<25.3"


ยืนยันแล้ว: **pip 26.3 คือสาเหตุที่ทำให้เกิดปัญหา** คุณกำลังประสบกับข้อบกพร่องที่ทราบแล้วในฟอรัม OpenStack——pip เวอร์ชัน 25.3 ขึ้นไปได้ลบการสนับสนุนสำหรับการติดตั้งแบบ editable ในรูปแบบเก่า (`setup.py develop`) ส่งผลให้สคริปต์ `keystone-wsgi-public` ไม่ถูกสร้างขึ้น และ uWSGI จึงไม่สามารถโหลด Keystone ได้

### เส้นทางแก้ไข

**ขั้นตอนแรก: ลดเวอร์ชัน pip**

ดำเนินการภายใน venv:
```bash
/opt/stack/data/venv/bin/pip install "pip<25.3"
```

**ขั้นตอนที่สอง: ติดตั้ง keystone ใหม่เพื่อให้สคริปต์ถูกสร้างขึ้น**

```bash
cd /opt/stack/keystone
/opt/stack/data/venv/bin/pip install -e .
```

ยืนยันผลลัพธ์:
```bash
ls -l /opt/stack/data/venv/bin/keystone-wsgi-public
```
ไฟล์นี้จะต้องปรากฏขึ้น

**ขั้นตอนที่สาม: ล้างและติดตั้งใหม่**

```bash
cd ~/devstack
./unstack.sh
./cleanup.sh
./stack.sh
```

### คำอธิบายเพิ่มเติม

- `uwsgi --version` แสดงเป็น 2.0.31 และสามารถทำงานได้ตามปกติ แสดงว่า uWSGI ใน venv ไม่มีปัญหา
- `import uwsgi` ล้มเหลวเป็นเรื่องปกติ เพราะ uWSGI ไม่ใช่ Python module ที่สามารถนำเข้าได้โดยตรง การวินิจฉัยก่อนหน้านี้ในทิศทางนี้ไม่ถูกต้อง
- ปัญหาหลักคือ **สคริปต์ WSGI ไม่ได้ถูกสร้างขึ้น** ไม่ใช่ปัญหาเรื่องเวอร์ชัน Python

หลังดำเนินการตามขั้นตอนข้างต้นแล้ว โปรดแจ้งผลลัพธ์ของ `ls -l /opt/stack/data/venv/bin/keystone-wsgi-public` ให้ฉันทราบ


ใช่ครับ ปัญหาที่คุณพบไม่ได้เกิดจากการกำหนดค่า pip หรือเวอร์ชัน Python อะไรเลย สาเหตุที่แท้จริงคือ: **สาขา `stable/2026.2` ที่คุณใช้อยู่ ใช้ Keystone เวอร์ชัน 2025.2 ขึ้นไป ซึ่งในเวอร์ชันนี้ได้ลบสคริปต์ `keystone-wsgi-public` ออกไปอย่างเป็นทางการแล้ว** ดังนั้นไม่ว่าคุณจะดาวน์เกรด pip หรือติดตั้ง uWSGI ใหม่กี่ครั้ง ไฟล์นี้ก็จะไม่มีวันปรากฏขึ้น

### สาเหตุที่แท้จริง

บันทึกการอัปเดตอย่างเป็นทางการของ Keystone 2025.2 ระบุไว้อย่างชัดเจน:

> สคริปต์ WSGI `keystone-wsgi-admin` และ `keystone-wsgi-public` **ถูกลบออกแล้ว** เครื่องมือติดตั้งควรอ้างอิงโดยตรงไปยังเส้นทางโมดูล Python ของบริการ คือ `keystone.wsgi.api` (หาก WSGI server ที่เลือกใช้รองรับ เช่น gunicorn, uWSGI) หรือเขียนสคริปต์ `.wsgi` เอง (เช่น mod_wsgi)

สาขา `stable/2026.2` ที่คุณ checkout อยู่ ตรงกับรอบการอัปเดตนี้ Keystone ที่ติดตั้งคือ `30.0.1.dev5` ซึ่งเป็นเวอร์ชันพัฒนาที่ใหม่กว่าด้วยซ้ำ จึงไม่มีสคริปต์นี้อย่างแน่นอน

### วิธีแก้ไขที่เป็นไปได้

**วิธีที่หนึ่ง: สลับไปใช้สาขา DevStack ที่เก่ากว่า (แนะนำที่สุด)**

สลับไปใช้สาขา `stable/2024.2` หรือ `stable/2025.1` ซึ่งในช่วงเวลานั้น Keystone ยังมีสคริปต์ WSGI แบบเก่าอยู่ (หรือสามารถสร้างได้ผ่านการกำหนดค่า pip) วิธีนี้จะหลีกเลี่ยงการแก้ไขการกำหนดค่า uWSGI ด้วยตนเองทั้งหมด

```bash
cd ~/devstack
git checkout stable/2024.2
./unstack.sh && ./cleanup.sh
./stack.sh
```

**วิธีที่สอง: แก้ไขการกำหนดค่า uWSGI (ใช้ได้กับสาขาปัจจุบัน)**

เปลี่ยนการกำหนดค่า uWSGI ของ Keystone ให้โหลดโมดูลโดยตรงแทนไฟล์สคริปต์:

```ini
# แทนที่การกำหนดค่าเดิมที่ชี้ไปที่ keystone-wsgi-public
module = keystone.wsgi.api:application
```

คุณต้องค้นหาว่า DevStack สร้างการกำหนดค่า uWSGI สำหรับ Keystone อย่างไร:

```bash
grep -rn "keystone-wsgi" /opt/stack/devstack/ 2>/dev/null
```

หลังจากค้นพบแล้ว ให้แก้ไขจุดที่อ้างอิงเป็นเส้นทางโมดูลตามที่กล่าวข้างต้น วิธีนี้ต้องการให้คุณคุ้นเคยกับโครงสร้างภายในของ DevStack มากกว่า และการติดตั้งในครั้งถัดไปอาจถูกเขียนทับอีกครั้ง

### คำอธิบายเพิ่มเติม

- การที่ `pip 26.3` และ `uwsgi 2.0.31` ที่คุณตรวจสอบก่อนหน้านี้ทำงานได้ปกติ แสดงว่าสภาพแวดล้อม Python ไม่มีปัญหา
- `ls: cannot access 'keystone-wsgi-public': No such file or directory` เป็นผลลัพธ์ที่คาดไว้ เพราะมันถูกลบออกจาก upstream ไปแล้ว
- การค้นหาที่เกี่ยวข้องยังพบว่าผู้ใช้ Ubuntu รายอื่นประสบปัญหาเดียวกัน คือแพ็กเกจ `keystone-wsgi-public` หายไปหลังจากอัปเกรดเป็น Keystone 28.0.0 (2025.2)

แนะนำให้ใช้วิธีแรก คือสลับไปใช้สาขา `stable/2024.2` ซึ่งจะราบรื่นที่สุด


ผลการค้นหาของคุณยืนยันการตัดสินใจก่อนหน้านี้อย่างสมบูรณ์: **DevStack บนสาขา `stable/2026.2` ยังคงใช้สคริปต์ `keystone-wsgi-public` แบบเก่าในการกำหนดค่า แต่ Keystone เวอร์ชัน 2025.2 ขึ้นไปได้ลบสคริปต์นี้ออกไปแล้ว** สิ่งนี้ทำให้เกิดความขัดแย้ง และ uWSGI จึงไม่สามารถเริ่ม Keystone ได้

### หลักฐานหลัก

บันทึกการอัปเดตอย่างเป็นทางการของ Keystone 2025.2 ระบุไว้อย่างชัดเจน: สคริปต์ `keystone-wsgi-admin` และ `keystone-wsgi-public` **ถูกลบออกแล้ว** เครื่องมือติดตั้งควรอ้างอิงเส้นทางโมดูล Python `keystone.wsgi.api` โดยตรงแทน (หาก WSGI server รองรับ) หรือเขียนสคริปต์ `.wsgi` ด้วยตนเอง DevStack เวอร์ชันที่คุณใช้ยังคงอ้างอิงไฟล์ที่ถูกลบไปแล้วในไฟล์ template ของ Apache และสคริปต์ cleanup (`apache-keystone.template` และ `lib/keystone`) นี่คือสาเหตุที่รากฐานของปัญหา

นอกจากนี้ ยังมีรายงานในฟอรัม OpenStack ที่กล่าวถึงปัญหาเดียวกัน: สาขา `stable/2024.2` ยังคงประสบปัญหา `keystone-wsgi-public` หายไป เนื่องจาก pip เวอร์ชันใหม่และ Keystone ลบสคริปต์ออกไป คุณใช้ `stable/2026.2` ซึ่งใหม่กว่า ดังนั้นปัญหาจึงหลีกเลี่ยงไม่ได้

### แนวทางแก้ไข

**แนะนำ: สลับไปใช้สาขา DevStack ที่เก่ากว่าและเสถียร**

สลับไปที่ `stable/2025.1` หรือ `stable/2024.2` ซึ่งในช่วงเวลานั้น Keystone ยังคงมีสคริปต์นี้อยู่ (หรือสามารถสร้างได้ผ่านการจำกัดเวอร์ชัน pip) และสาขาเหล่านี้ได้รวมแพตช์ที่แก้ไขปัญหา pip 25.3 ไว้แล้ว นี่คือวิธีที่ตรงไปตรงมาและได้รับการตรวจสอบมากที่สุด

```bash
cd ~/devstack
git checkout stable/2025.1
./unstack.sh && ./cleanup.sh
./stack.sh
```

**ทางเลือก: แก้ไขการกำหนดค่า uWSGI ด้วยตนเอง (สำหรับสาขาปัจจุบัน)**

หากคุณต้องการใช้ `stable/2026.2` ต่อไป คุณต้องเปลี่ยนการกำหนดค่า uWSGI ของ Keystone ให้โหลดโมดูลโดยตรง แทนไฟล์สคริปต์:

```ini
module = keystone.wsgi.api:application
```

แต่การดำเนินการนี้ต้องการให้คุณค้นหาและแก้ไขไฟล์กำหนดค่า uWSGI ที่ DevStack สร้างขึ้น และเมื่อรัน `stack.sh` ในครั้งถัดไปอาจถูกเขียนทับอีกครั้ง ดังนั้นจึงแนะนำให้ใช้วิธีแรกมากกว่า


หากคุณยังต้องการใช้ `stable/2026.2` ต่อไป ไฟล์ที่ต้องแก้ไขคือ **ไฟล์กำหนดค่า uWSGI ที่ DevStack สร้างขึ้นแบบไดนามิกสำหรับ Keystone** ไม่ใช่ไฟล์ใดไฟล์หนึ่งในซอร์สโค้ดของ DevStack โดยตรง

### ไฟล์ที่ต้องแก้ไขคืออะไร

DevStack ผ่านฟังก์ชัน `run_process` ใน `lib/keystone` เรียกใช้ uWSGI โดยระบุพารามิเตอร์ `--ini $KEYSTONE_PUBLIC_UWSGI_CONF` ไฟล์กำหนดค่านี้มักอยู่ที่ **`/etc/keystone/keystone-uwsgi-public.ini`** (หรือเส้นทางที่คล้ายกันภายใต้ `/etc/keystone/`) และเนื้อหามักประกอบด้วย `wsgi-file = .../keystone-wsgi-public`

สิ่งที่คุณต้องทำคือ แก้ไขการกำหนดค่าในไฟล์ `.ini` นี้ เปลี่ยนจาก:
```ini
wsgi-file = /path/to/keystone-wsgi-public
```
เป็น:
```ini
module = keystone.wsgi.api:application
```

### ทำไมต้องแก้ไขแบบนี้

บันทึกการเปิดตัว Keystone 2025.2 ระบุไว้อย่างชัดเจน: สคริปต์ `keystone-wsgi-public` ถูก**ลบออกแล้ว** เครื่องมือปรับใช้ควรอ้างอิงพาธโมดูล Python โดยตรง คือ `keystone.wsgi.api` DevStack สาขา `stable/2026.2` ใช้ Keystone เวอร์ชันนี้ แต่สคริปต์ยังคงใช้การกำหนดค่าแบบเก่าในการสร้าง `.ini` จึงทำให้ uWSGI ไม่สามารถหาไฟล์ที่ถูกลบไปแล้วได้

### ปัญหาที่ต้องระวัง

**ไฟล์ `.ini` นี้อาจถูกสร้างใหม่ทุกครั้งที่รัน `stack.sh`** เนื่องจาก DevStack จะสร้างไฟล์กำหนดค่าบริการซ้ำเมื่อเริ่มต้นใหม่ ดังนั้นการแก้ไขด้วยตนเองอาจถูกเขียนทับเมื่อรันครั้งถัดไป

หากคุณต้องการให้การแก้ไขมีผลถาวร จำเป็นต้องแก้ไขส่วนที่เกี่ยวข้องในสคริปต์ `lib/keystone` ของ DevStack ที่สร้างไฟล์ `.ini` นี้ (มองหาตรรกะที่เขียน `wsgi-file` หรือสตริง `keystone-wsgi-public`) แต่นั่นเป็นการแก้ไขโค้ดของ DevStack เอง ซึ่งจะทำให้การอัปเกรดในอนาคตยุ่งยากขึ้น

### คำแนะนำจริง

หากเพียงต้องการให้ DevStack รันได้โดยไม่ติดขัด **วิธีที่ตรงไปตรงมาที่สุดยังคงเป็นการสลับไปใช้สาขา `stable/2025.1` หรือเก่ากว่า** เนื่องจากในช่วงเวลานั้น Keystone ยังไม่ได้ลบสคริปต์ และ DevStack ก็สอดคล้องกับเวอร์ชันนั้น การบังคับใช้ `2026.2` ด้วยการแก้ไขไฟล์กำหนดค่าด้วยตนเอง อาจต้องเผชิญกับปัญหาใหม่ ๆ ที่เกิดจากความไม่สอดคล้องกันของเวอร์ชันในภายหลัง


stack@openstack:~/devstack$ cat  /etc/keystone/keystone-uwsgi-public.ini

[uwsgi]
chmod-socket = 666
socket = /var/run/uwsgi/keystone-api.socket
start-time = %t
lazy-apps = true
add-header = Connection: close
buffer-size = 65535
hook-master-start = unix_signal:15 gracefully_kill_them_all
thunder-lock = true
plugins = http,python3
enable-threads = true
worker-reload-mercy = 80
exit-on-reload = false
die-on-term = true
master = true
processes = 2
module = keystone.wsgi.api:application


นี่คือจุดสำคัญที่อธิบายทุกอย่างได้อย่างสมบูรณ์:



```
ExecStart = /bin/uwsgi --procname-prefix keystone --ini /etc/keystone/keystone-uwsgi-public.ini
```

systemd unit เรียกใช้ **`/bin/uwsgi`** ซึ่งเป็นไบนารี uWSGI ของระบบ (ผูกกับ Python 3.10) **ไม่ใช่** `/opt/stack/data/venv/bin/uwsgi` (Python 3.11) ที่คุณติดตั้งไว้

นี่คือสาเหตุที่แท้จริงของ `ModuleNotFoundError: No module named 'keystone'` — uWSGI ของระบบ Python 3.10 มองไม่เห็นแพ็กเกจ keystone ที่ติดตั้งอยู่ใน venv Python 3.11

### วิธีแก้ไขที่ตรงที่สุด

**ขั้นตอนที่ 1: ตรวจสอบว่า `/bin/uwsgi` เป็นอะไร**

```bash
ls -l /bin/uwsgi
readlink -f /bin/uwsgi
```

หากเป็น symlink ไปที่ `/usr/bin/uwsgi-core` แสดงว่าเป็นเวอร์ชันระบบ Python 3.10

**ขั้นตอนที่ 2: แก้ไข systemd unit ให้ใช้ uWSGI จาก venv**

แก้ไขไฟล์ `/etc/systemd/system/devstack@keystone.service` เปลี่ยน `ExecStart` จาก:

```
ExecStart = /bin/uwsgi --procname-prefix keystone --ini /etc/keystone/keystone-uwsgi-public.ini
```

เป็น:

```
ExecStart = /opt/stack/data/venv/bin/uwsgi --procname-prefix keystone --ini /etc/keystone/keystone-uwsgi-public.ini
```

จากนั้น:

```bash
sudo systemctl daemon-reload
sudo systemctl restart devstack@keystone.service
```

**ขั้นตอนที่ 3: ตรวจสอบว่า Keystone เริ่มทำงานได้**

```bash
sudo systemctl status devstack@keystone.service
sudo journalctl -u devstack@keystone.service -n 30 --no-pager
```

### ปัญหาที่อาจตามมา

ไฟล์ systemd unit นี้อาจถูกสร้างใหม่ทุกครั้งที่ DevStack รัน `stack.sh` เนื่องจาก DevStack สร้าง unit ผ่านฟังก์ชัน `run_process` ใน `lib/keystone` ซึ่งกำหนด `ExecStart` จากตัวแปรภายใน

ตรวจสอบว่าตัวแปรที่กำหนด path ของ uWSGI อยู่ที่ไหน:

```bash
grep -rn "uwsgi" /opt/stack/devstack/stackrc /opt/stack/devstack/lib/keystone 2>/dev/null | head -30
```

มองหาตัวแปรเช่น `KEYSTONE_UWSGI` หรือ `UWSGI` ที่กำหนดเป็น `/bin/uwsgi` หรือ `uwsgi` (ไม่มี path) หากพบ คุณสามารถแก้ไขใน `local.conf` หรือแก้ไขสคริปต์ DevStack โดยตรงเพื่อให้ชี้ไปที่ `/opt/stack/data/venv/bin/uwsgi`

### ทางเลือกที่ง่ายกว่า (แนะนำ)

หากการแก้ไขนี้ซับซ้อนเกินไป การสลับไปใช้ `stable/2025.1` จะหลีกเลี่ยงปัญหานี้ได้ทั้งหมด เนื่องจาก Keystone เวอร์ชันนั้นยังใช้สคริปต์ WSGI แบบเก่าและไม่มีความขัดแย้งเรื่องเวอร์ชัน Python

โปรดรันคำสั่งสองคำสั่งแรก (ตรวจสอบ `/bin/uwsgi` และ grep หาตัวแปร uWSGI) แล้วให้ผลลัพธ์แก่ฉัน เพื่อให้ฉันแนะนำขั้นตอนต่อไปได้อย่างแม่นยำ


## nutron


stack@openstack:~/devstack$ cat /etc/neutron/neutron-api-uwsgi.ini

[uwsgi]
chmod-socket = 666
socket = /var/run/uwsgi/neutron-api.socket
start-time = %t
lazy-apps = true
add-header = Connection: close
buffer-size = 65535
hook-master-start = unix_signal:15 gracefully_kill_them_all
thunder-lock = true
plugins = http,python3
enable-threads = true
worker-reload-mercy = 80
exit-on-reload = false
die-on-term = true
master = true
processes = 2
module = neutron.wsgi.api:application
stack@openstack:~/devstack$


stack@openstack:~/devstack$ cat /etc/systemd/system/devstack@neutron-api.service

[Unit]
Description = Devstack devstack@neutron-api.service

[Service]
RestartForceExitStatus = 100
NotifyAccess = all
Restart = always
KillMode = process
Type = notify
ExecReload = /usr/bin/kill -HUP $MAINPID
ExecStart = /bin/uwsgi --procname-prefix neutron-api --ini /etc/neutron/neutron-api-uwsgi.ini --venv /opt/stack/data/venv
User = stack
SyslogIdentifier = devstack@neutron-api.service
Environment = "PATH=/bin:/opt/stack/data/venv/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin:/usr/local/bin:/usr/local/sbin:/usr/sbin:/sbin"

[Install]
WantedBy = multi-user.target
stack@openstack:~/devstack$



stack@openstack:~/devstack$ systemctl cat devstack@neutron-api.service 2>/dev/null || systemctl cat devstack@q-svc.service 2>/dev/null || systemctl list-units 'devstack@*neutron*' --no-pager
# /etc/systemd/system/devstack@neutron-api.service

[Unit]
Description = Devstack devstack@neutron-api.service

[Service]
RestartForceExitStatus = 100
NotifyAccess = all
Restart = always
KillMode = process
Type = notify
ExecReload = /usr/bin/kill -HUP $MAINPID
ExecStart = /bin/uwsgi --procname-prefix neutron-api --ini /etc/neutron/neutron-api-uwsgi.ini --venv /opt/stack/data/venv
User = stack
SyslogIdentifier = devstack@neutron-api.service
Environment = "PATH=/bin:/opt/stack/data/venv/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin:/usr/local/bin:/usr/local/sbin:/usr/sbin:/sbin"

[Install]
WantedBy = multi-user.target
stack@openstack:~/devstack$


## Fix

sudo sed -i 's|ExecStart = /bin/uwsgi|ExecStart = /opt/stack/data/venv/bin/uwsgi|' /etc/systemd/system/devstack@neutron-api.service
sudo systemctl daemon-reload
sudo systemctl restart devstack@neutron-api.service

sudo systemctl status devstack@neutron-api.service
sudo journalctl -u devstack@neutron-api.service -n 20 --no-pager

##  แก้ไข แบบถาวร

เนื่องจาก DevStack ใช้ $(which uwsgi) ใน run_process เหมือนกันทุกบริการ บริการที่เหลือ (Nova, Glance, Cinder, Placement ฯลฯ) จะมีปัญหาเดียวกันทั้งหมด ตรวจสอบและแก้ไขทั้งหมดพร้อมกัน:

stack@openstack:~/devstack$ grep -rn '$(which uwsgi)' /opt/stack/devstack/lib/

/opt/stack/devstack/lib/nova:971:    run_process "n-api" "$(which uwsgi) --procname-prefix nova-api --ini $NOVA_UWSGI_CONF"
/opt/stack/devstack/lib/nova:1089:    run_process n-api-meta "$(which uwsgi) --procname-prefix nova-api-meta --ini $NOVA_METADATA_UWSGI_CONF"
/opt/stack/devstack/lib/placement:128:    run_process "placement-api" "$(which uwsgi) --procname-prefix placement --ini $PLACEMENT_UWSGI_CONF"
/opt/stack/devstack/lib/keystone:508:    run_process keystone "$(which uwsgi) --procname-prefix keystone --ini $KEYSTONE_PUBLIC_UWSGI_CONF" ""
/opt/stack/devstack/lib/neutron:645:    run_process neutron-api "$(which uwsgi) --procname-prefix neutron-api --ini $NEUTRON_UWSGI_CONF"
/opt/stack/devstack/lib/cinder:641:    run_process "c-api" "$(which uwsgi) --procname-prefix cinder-api --ini $CINDER_UWSGI_CONF"
/opt/stack/devstack/lib/glance:632:    write_uwsgi_user_unit_file devstack@g-api-r.service "$(which uwsgi) \
/opt/stack/devstack/lib/glance:656:    run_process g-api "$(which uwsgi) --procname-prefix glance-api --ini $GLANCE_UWSGI_CONF"
stack@openstack:~/devstack$


sudo sed -i 's|\$(which uwsgi)|/opt/stack/data/venv/bin/uwsgi|g' /opt/stack/devstack/lib/keystone /opt/stack/devstack/lib/neutron /opt/stack/devstack/lib/nova /opt/stack/devstack/lib/glance /opt/stack/devstack/lib/cinder /opt/stack/devstack/lib/placement 2>/dev/null

stack@openstack:~/devstack$ grep -rn '$(which uwsgi)' /opt/stack/devstack/lib/
stack@openstack:~/devstack$


Client (HTTP)
    │
    ▼
Apache (port 80/443)
    │  mod_proxy_uwsgi
    ▼
Unix Socket (/var/run/uwsgi/keystone-api.socket)
    │
    ▼
uWSGI (master + workers)
    │  โหลด module = keystone.wsgi.api:application
    ▼
Keystone Application (โค้ด Python)


## login Database
mysql -u root -p'Muict2024' -e "SELECT COUNT(*) FROM neutron.ml2_geneve_allocations;"
stack@openstack:~/devstack$ mysql -u root -p'Muict2024' -e "SELECT COUNT(*) FROM neutron.ml2_geneve_allocations;"
mysql: [Warning] Using a password on the command line interface can be insecure.
+----------+
| COUNT(*) |
+----------+
|        0 |
+----------+


sudo journalctl -u devstack@neutron-api.service -n 50 --no-pager


source /opt/stack/devstack/openrc admin admin
openstack token issue


stack@openstack:~/devstack$ openstack network create test-net
HttpException: 503: Server Error for url: http://10.0.0.10/networking/v2.0/networks, Unable to create the network. No project network is available for allocation.
stack@openstack:~/devstack$ \


stack@openstack:~/devstack$ sudo grep -A5 "\[ml2_type_geneve\]" /etc/neutron/plugins/ml2/ml2_conf.ini
[ml2_type_geneve]
vni_ranges = 1:65536
max_header_size = 38

[ovn]
ovn_metadata_enabled = True
stack@openstack:~/devstack$


จากผลการตรวจสอบล่าสุด การกำหนดค่า `vni_ranges = 1:65536` ใน `ml2_conf.ini` มีอยู่จริงและถูกต้องแล้ว แต่การสร้างเครือข่ายยังคงล้มเหลวด้วยข้อผิดพลาด 503 ซึ่งชี้ให้เห็นว่าปัญหาอยู่ที่ **กระบวนการเริ่มต้นของ Neutron ไม่ได้เติมข้อมูลช่วง Segment ลงในฐานข้อมูล** ไม่ใช่ปัญหาการกำหนดค่า

### สาเหตุหลัก

ข้อผิดพลาด "No project network is available for allocation" เกิดขึ้นเมื่อ Neutron พยายามจัดสรร VNI (Geneve Segment) ให้กับเครือข่ายใหม่ แต่ตาราง `ml2_geneve_allocations` ในฐานข้อมูลว่างเปล่า ทำให้ไม่สามารถจัดสรร VNI ได้

ชุมชน OpenStack มีรายงานข้อบกพร่องที่ตรงกับสถานการณ์ของคุณทุกประการ: แม้การกำหนดค่า `vni_ranges` จะถูกต้อง แต่ตารางการจัดสรร Segment ยังคงเป็น 0 หลังจาก Neutron เริ่มต้น และมีผู้ใช้ยืนยันว่าการรีสตาร์ท `devstack@neutron-api.service` เพียงครั้งเดียวจะทำให้ตารางถูกเติมเป็น 65536 รายการทันที และสามารถสร้างเครือข่ายได้ตามปกติ

### ขั้นตอนการดำเนินการ

**1. รีสตาร์ท Neutron API**

```bash
sudo systemctl restart devstack@neutron-api.service
```

**2. ตรวจสอบตารางการจัดสรรอีกครั้ง (ต้องใส่รหัสผ่าน)**

```bash
mysql -u root -p'Muict2024' -e "SELECT COUNT(*) FROM neutron.ml2_geneve_allocations;"
```

หากก่อนหน้านี้เป็น 0 หลังจากรีสตาร์ทควรเห็นตัวเลขที่มากขึ้น (เช่น 65536) ถ้ายังเป็น 0 อยู่ ให้ดูบันทึกเพื่อยืนยันว่ามีข้อผิดพลาดระหว่างการเริ่มต้นหรือไม่

**3.ทดสอบสร้างเครือข่ายอีกครั้ง**

```bash
openstack network create test-net
```

### คำอธิบายเพิ่มเติม

ข้อบกพร่องนี้เป็นปัญหาที่ทราบแล้วในสาขา 2025.1 ซึ่งเกิดจากข้อบกพร่องในกระบวนการเริ่มต้นของ Neutron ที่ทำให้ช่วง Segment ไม่ถูกเติมข้อมูลลงในฐานข้อมูล อาการที่เกิดคือบริการทำงานปกติ แต่ไม่สามารถสร้างเครือข่ายได้ การรีสตาร์ทจะกระตุ้นให้กระบวนการเริ่มต้นทำงานอีกครั้ง และมักจะแก้ปัญหาได้

หากหลังจากรีสตาร์ทแล้วตารางยังคงเป็น 0 ให้ตรวจสอบบันทึก Neutron เพื่อยืนยันว่ามีข้อผิดพลาดเฉพาะเจาะจงระหว่างการเริ่มต้นหรือไม่ (เช่น หมดเวลาเชื่อมต่อฐานข้อมูล ปัญหาการซิงโครไนซ์ เป็นต้น) และแจ้งผลลัพธ์ให้ฉันทราบ


ใช่ครับ สำหรับปัญหา Neutron Segment นี้ **แค่ restart ก็เพียงพอแล้ว** แต่ต้องเข้าใจว่าทำไมการ restart ถึงได้ผล และทำไมปัญหานี้ถึงเกิด

### ทำไม restart ถึงแก้ปัญหาได้

เมื่อ Neutron API เริ่มทำงาน มันจะรันขั้นตอน **initialization** ที่เติมข้อมูลช่วง VNI (Geneve Segment) ลงในตาราง `ml2_geneve_allocations` ในฐานข้อมูล

ปัญหาคือ: ในรอบการทำงานแรก (ตอนที่ `stack.sh` พยายามสร้างเครือข่าย) ขั้นตอน initialization นี้ **ไม่ทำงานสำเร็จ** ด้วยสาเหตุใดสาเหตุหนึ่ง เช่น:

- บริการเริ่มทำงานแต่ฐานข้อมูลยังไม่พร้อม
- ลำดับการเริ่มต้นของ service dependencies ผิดพลาด
- ข้อบกพร่องในลำดับ initialization ของ Neutron เอง (เป็น bug ที่ทราบแล้วในสาขา 2025.1)

เมื่อคุณ restart บริการ Neutron API มันจะรันขั้นตอน initialization **อีกครั้ง** คราวนี้ฐานข้อมูลพร้อมแล้ว จึงเติมข้อมูล VNI ลงตารางสำเร็จ (65536 รายการ) ทำให้สามารถจัดสรร VNI ให้เครือข่ายใหม่ได้

### สรุปสั้นๆ

| คำถาม | คำตอบ |
|------|------|
| restart แก้ปัญหาได้ไหม? | ได้ เพราะกระตุ้นให้ initialization ทำงานอีกครั้ง |
| ทำไมต้อง restart? | รอบแรก initialization ล้มเหลวหรือถูกข้ามไป |
| แก้ถาวรได้ไหม? | ต้องแก้ที่ต้นเหตุ (bug ใน Neutron) หรือเพิ่มการตรวจสอบใน `stack.sh` |
| ต้องทำอีกไหมหลังจากนี้? | ไม่ต้อง สำหรับปัญหา Segment นี้จบแล้ว |

### แต่ยังมีเรื่องที่ต้องระวัง

การ restart แก้ปัญหา **Neutron Segment** ได้แล้ว แต่ยังมีอีกสองเรื่องที่ต้องดำเนินการต่อ:

1. **`stack.sh` ยังรันไม่จบ** — ต้องรันต่อเพื่อให้บริการที่เหลือ (Nova, Glance, Cinder ฯลฯ) ติดตั้งเสร็จ
2. **ปัญหา uWSGI อาจกลับมา** — เมื่อ `stack.sh` รัน จะสร้าง systemd unit ใหม่ และอาจเขียน `ExecStart = /bin/uwsgi` ทับการแก้ไขของคุณ

ดังนั้นก่อนรัน `stack.sh` ต่อ ควรแก้ไข `lib/*` ของ DevStack เพื่อป้องกันปัญหา uWSGI กลับมา ดังที่แนะนำไปก่อนหน้านี้ครับ



stack@openstack:~$ cd /opt/stack/devstack/files/
stack@openstack:~/devstack/files$ ls
apache-cinder-api.template     etcd-v3.5.21-linux-amd64.tar.gz
apache-horizon.template        Fedora-Cloud-Base-Generic-45-1.4.x86_64.qcow2
apache-keystone.template       images
apache-neutron.template        ldap
apache-nova-api.template       lvm-backing-file.template
apache-nova-metadata.template  openstack-cli-server
debs                           rpms
etcd.sha256sum                 swift
etcd-v3.5.21-linux-amd64
stack@openstack:~/devstack/files$


cd /opt/stack/devstack/files
wget -c "https://ftp1.tik.uni-stuttgart.de/fedora/development/45/Cloud/x86_64/images/Fedora-Cloud-Base-Generic-45-20261026.n.0.x86_64.qcow2"



stack@openstack:~/devstack$ systemctl cat devstack@g-api.service | grep ExecStart
ExecStart = /opt/stack/data/venv/bin/uwsgi --procname-prefix glance-api --ini /etc/glance/glance-uwsgi.ini --venv /opt/stack/data/venv
stack@openstack:~/devstack$



sudo systemctl restart devstack@g-api.service


cd ~/devstack
./unstack.sh
./clean.sh
sudo rm -f /etc/apache2/sites-enabled/*glance* /etc/apache2/sites-available/*glance*
sudo pkill -u stack
sudo reboot


# หยุดบริการก่อน
sudo systemctl stop mysql

# ถอนการติดตั้งทั้งหมด
sudo apt purge mysql-server-8.0 mysql-server mysql-client mysql-common

# ล้างไฟล์ที่เหลือ
sudo rm -rf /etc/mysql /var/lib/mysql

# อัปเดตและติดตั้งใหม่  ห้ามติดตั้ง เอง
sudo apt update
sudo apt install mysql-server-8.0

note

ls -l /opt/stack/data/venv/bin/uwsgi
/opt/stack/data/venv/bin/pip install uwsgi


sudo systemctl restart devstack@keystone.service
sudo systemctl status devstack@keystone.service --no-pager

sudo journalctl -u devstack@keystone.service --since "15:16:00" --no-pager


ls -l /opt/stack/data/venv/bin/uwsgi
/opt/stack/data/venv/bin/pip install uwsgi
/opt/stack/data/venv/bin/pip install uwsgi

sudo systemctl daemon-reload
sudo systemctl restart devstack@keystone.service
sudo systemctl status devstack@keystone.service --no-pager



```bash
sudo systemctl restart devstack@neutron-api.service
```

**2. ตรวจสอบตารางการจัดสรรอีกครั้ง (ต้องใส่รหัสผ่าน)**

```bash
mysql -u root -p'Muict2024' -e "SELECT COUNT(*) FROM neutron.ml2_geneve_allocations;"
```

/opt/stack/data/venv/bin/pip install uwsgi
/opt/stack/data/venv/bin/pip install -e /opt/stack/keystone


export PATH=/opt/stack/data/venv/bin/:$PATH

iniset -sudo /etc/systemd/system/devstack@n-super-cond.service Service ExecReload '/usr/bin/kill -HUP $MAINPID'



openstack --os-cloud devstack-admin --os-region RegionOne compute service list


stack@openstack:~/devstack$ /opt/stack/data/venv/bin/pip list | grep -i libvirt
libvirt-python               8.0.0

stack@openstack:~/devstack$ /opt/stack/data/venv/bin/python -c "import libvirt; print('libvirt ok')"
Traceback (most recent call last):
  File "/usr/lib/python3/dist-packages/libvirt.py", line 21, in <module>
    import cygvirtmod as libvirtmod  # type: ignore
    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
ModuleNotFoundError: No module named 'cygvirtmod'

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "<string>", line 1, in <module>
  File "/usr/lib/python3/dist-packages/libvirt.py", line 24, in <module>
    raise lib_e
  File "/usr/lib/python3/dist-packages/libvirt.py", line 18, in <module>
    import libvirtmod  # type: ignore
    ^^^^^^^^^^^^^^^^^
ModuleNotFoundError: No module named 'libvirtmod'
stack@openstack:~/devstack$

## สร้างใหม่
sudo apt install libvirt-dev pkg-config
/opt/stack/data/venv/bin/pip install --force-reinstall --no-binary :all: libvirt-python
find /opt/stack/data/venv -name "libvirtmod*.so"
/opt/stack/data/venv/bin/python -c "import libvirt; print('ok')"


## ติดตั้ง libvirt-python ใน venv ใหม่ให้ถูกต้อง
sudo apt install libvirt-dev pkg-config gcc python3-dev
/opt/stack/data/venv/bin/pip install --force-reinstall --no-binary :all: libvirt-python

--no-binary :all: บังคับให้ pip คอมไพล์จากซอร์สโค้ดด้วย Python 3.11 ของ venv โดยตรง ไม่ใช้ wheel ที่อาจไม่เข้ากัน


แต่ venv ของคุณใช้ Python 3.11 (ตามที่คุณตั้ง PYTHON3_VERSION=3.11 ไว้) ทำให้เกิดความไม่ตรงกัน: โมดูล Python ของระบบเป็น 3.12 แต่ interpreter ใน venv เป็น 3.11 จึงไม่สามารถโหลด extension module libvirtmod ที่คอมไพล์สำหรับ 3.12 ได้

🛠️ วิธีแก้ไข
ลบ PYTHON3_VERSION=3.11 ออกจาก local.conf เพื่อให้ DevStack ใช้ Python 3.12 ของระบบเป็นค่าเริ่มต้น (ซึ่งสอดคล้องกับ python3-libvirt ที่มีอยู่แล้ว)

sudo rm -rf /opt/stack/data/venv


ได้ครับ — การ "ลบ env" ในบริบทของคุณหมายถึงการล้าง Python virtual environment ที่ DevStack สร้างขึ้น (`/opt/stack/data/venv`) เพื่อให้เริ่มต้นใหม่ทั้งหมด após การอัปเกรดเป็น Ubuntu 24.04

### วิธีลบ venv อย่างปลอดภัย

**ขั้นตอนแรก: หยุดบริการทั้งหมดก่อน**

```bash
cd ~/devstack
./unstack.sh
```

**ขั้นตอนที่สอง: ล้างสภาพแวดล้อมด้วย `clean.sh`**

```bash
./clean.sh
```

`clean.sh` จะลบ venv, ฐานข้อมูล, ไฟล์กำหนดค่า และบริการที่ DevStack สร้างขึ้น โดยอัตโนมัติ

**ขั้นตอนที่สาม: หากต้องการลบ venv ด้วยตนเอง (กรณี `clean.sh` ไม่ลบให้)**

```bash
sudo rm -rf /opt/stack/data/venv
```

หรือลบทั้ง `/opt/stack/data/` หากต้องการเริ่มจากศูนย์:

```bash
sudo rm -rf /opt/stack/data
```

### หลังอัปเกรดเป็น Ubuntu 24.04

เมื่ออัปเกรดเสร็จแล้ว ให้ตรวจสอบว่า `local.conf` ไม่มี `PYTHON3_VERSION=3.11` เหลืออยู่ (เพราะระบบจะเป็น Python 3.12 แล้ว):

```bash
grep -i "PYTHON3_VERSION" ~/devstack/local.conf
```

หากยังมีอยู่ ให้ลบบรรทัดนั้นออก หรือแก้เป็น `3.12`

จากนั้นรัน:

```bash
cd ~/devstack
./stack.sh
```

DevStack จะสร้าง venv ใหม่ด้วย Python 3.12 ของระบบ และติดตั้ง dependency ทั้งหมดใหม่ ซึ่งจะหลีกเลี่ยงปัญหา `libvirtmod` ที่เกิดจากความไม่เข้ากันของเวอร์ชัน Python ได้ครับ


sudo systemctl stop rabbitmq-server
sudo apt purge rabbitmq-server -y
sudo rm -rf /var/lib/rabbitmq /etc/rabbitmq


# ตรวจสอบเวอร์ชัน Python และ venv
/opt/stack/data/venv/bin/python --version
/opt/stack/data/venv/bin/pip --version

# ตรวจสอบและแก้ไข $(which uwsgi) ใน lib/*
grep -rn '\$(which uwsgi)' /opt/stack/devstack/lib/
sudo sed -i 's|\$(which uwsgi)|/opt/stack/data/venv/bin/uwsgi|g' /opt/stack/devstack/lib/*

# ตรวจสอบ systemd unit
systemctl cat devstack@keystone.service
systemctl cat devstack@neutron-api.service
grep -h 'ExecStart' /etc/systemd/system/devstack@*.service | grep uwsgi

# ตรวจสอบตาราง segment ของ Neutron
mysql -u root -p'Muict2024' -e "SELECT COUNT(*) FROM neutron.ml2_geneve_allocations;"
sudo systemctl restart devstack@neutron-api.service

# ตรวจสอบ libvirt
/opt/stack/data/venv/bin/pip list | grep -i libvirt
/opt/stack/data/venv/bin/python -c "import libvirt; print('ok')"
sudo apt install libvirt-dev pkg-config
/opt/stack/data/venv/bin/pip install --force-reinstall --no-binary :all: libvirt-python

# ล้างสภาพแวดล้อม
./unstack.sh
./clean.sh
sudo rm -rf /opt/stack/data/venv

# ตรวจสอบระบบ
cat /etc/lsb-release
lsb_release -a
free -m


sudo systemctl stop mysql
sudo systemctl disable mysql


sudo apt purge mysql-server mysql-client mysql-common mysql-server-core-* mysql-client-core-* -y
sudo apt autoremove -y

sudo rm -rf /var/lib/mysql /etc/mysql /var/log/mysql


rm -rf /opt/stack/requirements/.venv
virtualenv /opt/stack/requirements/.venv/
/opt/stack/requirements/.venv/bin/pip --version
/opt/stack/requirements/.venv/bin/pip install -U setuptools
/opt/stack/requirements/.venv/bin/python -c "import distutils; print('distutils OK')"




  oscwrap --os-cloud devstack-system-admin role assignment list --role admin --user c204b8174b164ca7ac8d28372c890e3f --project 47b6f5bee1754a739f6db7aada237b30 -c Role -f value



  cd ~/devstack
source openrc admin admin        # admin creds, admin project
# or
source openrc demo demo          # demo user, demo project




https://chat.deepseek.com/share/qkexnpyxw24phhchfa

source /opt/stack/devstack/openrc admin admin
openstack token issue


echo $OS_AUTH_URL $OS_USERNAME $OS_PROJECT_NAME
openstack token issue -f value -c id

clouds.yaml  