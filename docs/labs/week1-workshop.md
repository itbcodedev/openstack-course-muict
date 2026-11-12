---
title: "Workshop 1/3 Basic Openstack operation"
---

!!! abstrtact "เอกสาร Download สำหรับรายงาน"
    1. [เอกสาร ตัวอย่าง pdf](./Workshop%201_example.pdf) 
    2. [เอกสาร ตันฉบับ docs](./Workshop%201.docx)

- คะแนน  50 คะแนน
- ระยะเวลา 45-60 นาที

คำสั่งสำหรับ copy paste

```
source ~/devstack/openrc admin admin
openstack --version
openstack compute service list -c Binary -c Host -c State
openstack network agent list -c "Agent Type" -c Host -c Alive -c State
```

```
source ~/devstack/openrc demo demo
ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519 -q
openstack keypair create --public-key ~/.ssh/id_ed25519.pub demo-key

openstack security group rule create --proto icmp default

openstack security group rule create --proto tcp --dst-port 22 default

```

```
openstack server create --flavor m1.tiny --image cirros-0.6.3-x86_64-disk \
  --network private --key-name demo-key --wait vm01
```

```
FIP=$(openstack floating ip create public -f value -c floating_ip_address)

openstack server add floating ip vm01 "${FIP}"

openstack server list -c Name -c Status -c Networks

ping -c3 "${FIP}"
```

