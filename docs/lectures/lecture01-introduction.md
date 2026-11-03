# Lecture 1: Introduction to Cloud Computing & OpenStack

## Learning Objectives

- Understand cloud computing service and deployment models
- Know the history and ecosystem of OpenStack
- Identify the core OpenStack services and their roles

---

## What is Cloud Computing?

Cloud computing is the on-demand delivery of IT resources over the internet with pay-as-you-go pricing.

### Service Models

| Model | Description | Example |
|-------|-------------|---------|
| **IaaS** | Infrastructure as a Service — Rent VMs, storage, networks | OpenStack, AWS EC2 |
| **PaaS** | Platform as a Service — Deploy apps without managing infra | Heroku, OpenShift |
| **SaaS** | Software as a Service — Use applications over the web | Gmail, Salesforce |

### Deployment Models

- **Public Cloud** — Shared infrastructure (AWS, Azure, GCP)
- **Private Cloud** — Dedicated infrastructure for one organization
- **Hybrid Cloud** — Combination of public and private
- **Community Cloud** — Shared among organizations with common concerns

---

## OpenStack Overview

!!! quote "OpenStack Mission"
    To produce the ubiquitous Open Source cloud computing platform that meets the needs of public and private clouds regardless of size.

### History

- **2010** — Joint project by NASA and Rackspace
- **2012** — OpenStack Foundation formed
- **2023** — OpenInfra Foundation governs the project
- **Latest** — Antelope (2023.1), Bobcat (2023.2), Caracal (2024.1)

### Why OpenStack?

✅ Open source — no vendor lock-in  
✅ Large ecosystem of services  
✅ Used by thousands of organizations worldwide  
✅ Strong community support  
✅ API compatibility with AWS (in many services)

---

## Core Services Overview

| Service | Codename | Purpose |
|---------|----------|---------|
| Identity | **Keystone** | Authentication & authorization |
| Compute | **Nova** | Virtual machine provisioning |
| Networking | **Neutron** | Network connectivity as a service |
| Image | **Glance** | VM image storage & retrieval |
| Block Storage | **Cinder** | Persistent block volumes |
| Object Storage | **Swift** | Scalable object storage |
| Dashboard | **Horizon** | Web-based UI |
| Orchestration | **Heat** | Template-based resource deployment |
| Telemetry | **Ceilometer** | Metering & monitoring |

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    Horizon (Web UI)                      │
├─────────────────────────────────────────────────────────┤
│  OpenStack REST APIs  │  Python SDKs  │  CLI Clients    │
├─────────────────────────────────────────────────────────┤
│  Nova  │  Neutron  │  Cinder  │  Glance  │  Swift      │
├─────────────────────────────────────────────────────────┤
│  Message Queue (RabbitMQ)  │  Database (MariaDB)        │
├─────────────────────────────────────────────────────────┤
│  Hypervisor (KVM/QEMU)  │  Linux Bridge / OVS           │
└─────────────────────────────────────────────────────────┘
```

---

## Slides

📥 [Download Lecture 1 Slides (PDF)](../assets/slides/lecture01.pdf){ .md-button }

## Further Reading

- [OpenStack Documentation](https://docs.openstack.org/)
- [OpenInfra Foundation](https://openinfra.org/)
- [Cloud Computing NIST Definition (SP 800-145)](https://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-145.pdf)
