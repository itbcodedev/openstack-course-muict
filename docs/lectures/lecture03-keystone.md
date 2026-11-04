# การบรรยายที่ 3: Keystone — ระบบจัดการตัวตนและสิทธิ์

## วัตถุประสงค์การเรียนรู้

- เข้าใจความแตกต่างระหว่าง Authentication และ Authorization ใน OpenStack
- จัดการ Users, Projects, Roles และ Domains
- ทำงานกับ Service Catalog และ Endpoints

---

## Keystone คืออะไร?

Keystone คือ **บริการ Identity** ของ OpenStack ให้บริการ:

- **Authentication** — ยืนยันว่าคุณเป็นใคร
- **Authorization** — ตรวจสอบว่าคุณทำอะไรได้บ้าง
- **Service Catalog** — แสดงรายการบริการที่มีพร้อม Endpoints

---

## แนวคิดหลัก

```
┌─────────────────────────────────────────┐
│               Domain                     │
│  ┌─────────────────────────────────┐    │
│  │          Project (Tenant)        │   │
│  │  ┌───────────────────────────┐   │   │
│  │  │         User               │   │   │
│  │  │  Role: admin / member / reader │ │
│  │  └───────────────────────────┘   │   │
│  └─────────────────────────────────┘    │
└─────────────────────────────────────────┘
```

### Domain
- Namespace ระดับสูงสุดสำหรับ Users, Groups และ Projects
- รองรับ Multi-tenancy ระดับองค์กร
- Domain เริ่มต้น: `default`

### Project (เดิมช Tenant)
- คอนเทนเนอร์สำหรับทรัพยากร (VMs, Networks, Volumes)
- ผู้ใช้ได้รับสิทธิ์ผ่าน Roles ภายใน Project

### User
- เอนทิตีที่แทนบุคคลหรือ Service Account
- ยืนยันตัวตนผ่าน Password, Token หรือ External IdP

### Role
- กำหนดชุดสิทธิ์ภายใน Project
- Roles ทั่วไป: `admin`, `member`, `reader`

### Group
- ชุดของ Users
- สามารถกำหนด Roles ให้ Group ได้เพื่อง่ายต่อการจัดการ

---

## การยืนยันตัวตนด้วย Token

```
User credentials → Keystone
    ↓
Keystone ตรวจสอบ credentials
    ↓
คืน Token (UUID หรือ JWT)
    ↓
User นำ Token ไปแสดงต่อบริการอื่น ๆ
    ↓
บริการอื่น ๆ ขอ Keystone ตรวจสอบ Token
```

### รูปแบบ Token

| รูปแบบ | คำอธิบาย |
|------|-------------|
| **UUID** | String สุ่ม ตรวจสอบโดย Keystone ทุกครั้ง |
| **PKI/PKIz** | Self-validating ลายเซ็น Cryptographic |
| **Fernet** | Default ใน OpenStack สมัยใหม่ มี Payload ที่เข้ารหัส |

---

## Service Catalog และ Endpoints

Keystone เก็บ Catalog ของทุกบริการ OpenStack:

| ประเภท Endpoint | หน้าที่ |
|---------------|---------|
| **public** | API สำหรับภายนอก |
| **internal** | การสื่อสารระหว่างบริการภายใน |
| **admin** | การดำเนินการระดับ Admin |

ตัวอย่าง:
```yaml
Service: compute (nova)
  public:   https://cloud.muict.ac.th:8774/v2.1
  internal: http://10.0.0.11:8774/v2.1
  admin:    http://10.0.0.11:8774/v2.1
```

---

## ไฟล์ Policy

แต่ละบริการมี `policy.json` หรือ `policy.yaml` ที่จัด mapping ระหว่าง Roles กับ API Operations:

```yaml
# ตัวอย่าง: เฉพาะ admin สามารถสร้าง flavors ได้
"os_compute_api:os-flavor-manage:create": "rule:admin_api"
```

---

## Federated Identity

Keystone รองรับ External Identity Providers:

- **LDAP** — Active Directory, OpenLDAP
- **SAML 2.0** — Shibboleth, ADFS
- **OpenID Connect** — Google, Keycloak

ช่วยให้สามารถใช้ Single Sign-On (SSO) กับ OpenStack ได้

---

## คำสั่งใช้งานจริง

```bash
# แสดงรายการ Users
openstack user list

# สร้าง Project
openstack project create --description "โครงการนักศึกษา" student-project

# กำหนด Role ให้ User ใน Project
openstack role add --user alice --project student-project member

# ขอ Token
openstack token issue
```

---

## สไลด์การบรรยาย

📥 [ดาวน์โหลดสไลด์การบรรยายที่ 3 (PDF)](../assets/slides/Keystone%20ระบบจัดการตัวตนและสิทธิ์.pdf){ .md-button }

## แหล่งข้อมูลเพิ่มเติม

- [Keystone Administrator Guide](https://docs.openstack.org/keystone/latest/admin/)
- [RBAC ใน OpenStack](https://docs.openstack.org/keystone/latest/admin/auth-tokens.html)
