---
title: "นำเข้า Devstack Ubuntu image สู่ HyperV"
---

# นำเข้า Devstack Ubuntu image สู่ HyperV

## วัตถุประสงค์
1. เรียนรู้การ Import VM เข้ามาใน Hyper V
2. สร้าง internal network สำหรับ Devstack
3. ตรวจสอบ network eth0, eth1, eth2
4. ทดสอบการ SSH ยัง VM

!!!info
    Hyper-V คือ โปรแกรม Virtualization ของ Microsoft ที่สามารถใช้จำลองคอมพิวเตอร์เสมือนขึ้นมาบนคอมพิวเตอร์เครื่องเดียว เป็นฟีเจอร์ที่มีมาให้ในวินโดว์ Pro, Business, Enterprise และ Server Edition

## ตรวจสอบความพร้อม
- ให้เราพิมพ์คำว่า “windows features” ลงที่แถบ Search Bar ด้านล่าง จากนั้นกด Enter
![](./images/what-is-hyper-v-01.jpg)
- จะปรากฏหน้าต่าง Windows Features ขึ้นมา ซึ่งหน้านี้ เราสามารถเปิดหรือปิดฟีเจอร์ต่างๆ ของวินโดว์ได้ตามต้องการ ให้เราเลื่อนหาโฟลเดอร์ Hyper-V จากนั้น เลือก ที่กล่องด้านหน้า Hyper-V
![](./images/what-is-hyper-v-02.jpg)

## เอกสาร

[เอกสารประกอบการ นำเข้า Setup_DevStack_kvm.docx ](./Setup_DevStack_kvm.docx)


## สิ่งที่ควรรู้ การใช้งาน HyperV ใน Lab
### 1 ปัญหา copy Text จาก host ไปยัง VM linux
- การใช้งาน Web Dash board สามารถใช้งานได้โดยตรง เมื่อ เปิด VM ไม่มีปัญหาอะไร 
- แนะนำให้ SSH ไปยังเครื่อง VM จาก เครื่อง Windows จะสามารถทำได้ง่ายกว่า เมือต้องการใช้ Terminal

### 2 ปัญหาที่ เจอ การ login ไปยัง eth0 ให้เปลี่ยน eth0 จาก Static เป็น DHCP
- ในบางครั้ง ไม่สามารถ ssh ไปยัง eth0 ของ Server ได้ เนื่องจากมีการ Fix ip ไว้ใน file /etc/netplan/00-installer-config.yaml
    - ให้ทำการเปลี่ยน จาก โหมด Manual Fix ip เป็น โหมด DHCP เลือก ที่มุมขวาบน 
![](./images/lab01-12.png)
    - กดปุ่ม รูป Gear เป็นการ Setting
![](./images/lab01-13.png)
    - เลือก tab ipv4 >  Automatic (DHCP) > Apply
![](./images/lab01-14.png)
    - ปรับเป็น Enable
![](./images/lab01-15.png)
    - กด ดู Setting
![](./images/lab01-16.png)
    - จด ip ไว้เพื่อใช้สำหรับ login จาก windows (172.28.73.227)
![](./images/lab01-17.png)
    - เปิด Windows terminal ,  ssh ไปยัง  ip  user student,  password  muictP@ssw0rd
![](./images/lab01-18.png) 
    - ใช้งานได้ได้ตามปรกติ
![](./images/lab01-19.png)

# คำแนะนำ
- เนื่องจากเป็นการใช้งาน Linux ที่มีคำสั่ง หลายคำสั่ง แต่คำสั่ง เป็นการทำงานต่อเนื่องกัน เป็นลำดับขั้นตอน ดังนี้น ในระหว่างการทำ Lab ควรหากระดาษ หรือ note ใน ipad เพื่อจัดกลุ่มของคำสั่ง ด้วยความเข้าใจของตนเอง


[การทดลอง 2 Horizon Dashboard และ การจัดการ Project](./lab02-horizon.md)

# อ้างอิง
- การตั้งค่า network ใน  ubuntu [https://ubuntu.com/server/docs/explanation/networking/configuring-networks/](https://ubuntu.com/server/docs/explanation/networking/configuring-networks/)



