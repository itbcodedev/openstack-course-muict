# Lecture 2: OpenStack Architecture & Components

## Learning Objectives

- Understand the logical architecture of OpenStack
- Identify the role of each core component
- Learn about internal communication: APIs, message queues, and databases

---

## Logical Architecture

OpenStack follows a microservices architecture where each service:

- Runs independently
- Communicates via REST APIs
- Can scale horizontally

### Control Node vs. Compute Node

| Control Node | Compute Node |
|--------------|--------------|
| Runs API services, scheduler, database, message queue | Runs the hypervisor (VMs) |
| Keystone, Glance, Neutron Server, Nova API/Scheduler | Nova Compute, Neutron Agent |
| Usually 2+ for HA | Scales out based on workload |

---

## Internal Communication Flow

### Example: Launching a VM

```
User → Horizon / CLI
  ↓
Nova API → Keystone (authenticate token)
  ↓
Nova Scheduler (picks compute host)
  ↓
Message Queue → Nova Compute on selected host
  ↓
Glance (fetch image) + Neutron (allocate network)
  ↓
Libvirt/KVM creates VM on hypervisor
```

---

## Key Technologies

### Message Queue: RabbitMQ / AMQP

- Services communicate asynchronously via a message broker
- Ensures loose coupling between components
- Default: RabbitMQ, also supports Qpid and ZeroMQ

### Database: MariaDB / MySQL

- Stores persistent state for all services
- Each service has its own database schema
- Galera Cluster for HA deployments

### WSGI Server

- Python web services use WSGI (e.g., Apache mod_wsgi, uWSGI)
- Handles HTTP requests and routes to service code

---

## Service Components Breakdown

### Nova (Compute)

| Component | Role |
|-----------|------|
| nova-api | Receives and routes API requests |
| nova-scheduler | Decides which host runs a VM |
| nova-compute | Manages VMs on a hypervisor |
| nova-conductor | Database access mediator |
| nova-console | VNC/SPICE console proxy |

### Neutron (Networking)

| Component | Role |
|-----------|------|
| neutron-server | API and plugin manager |
| L2 Agent (OVS/Linux Bridge) | Manages virtual switches |
| L3 Agent | Handles routing & NAT |
| DHCP Agent | Provides IP addresses |
| Metadata Agent | Supplies cloud-init data |

---

## Deployment Topologies

### All-in-One (DevStack)
- Single node — everything on one machine
- Good for learning and development

### Two-Node
- Control node + Compute node
- Minimal production-like setup

### Multi-Node HA
- Multiple controllers with keepalived/haproxy
- Many compute nodes
- Separate storage and network nodes

---

## Slides

📥 [Download Lecture 2 Slides (PDF)](../assets/slides/lecture02.pdf){ .md-button }

## Further Reading

- [OpenStack Architecture Guide](https://docs.openstack.org/arch-design/)
- [RabbitMQ Tutorials](https://www.rabbitmq.com/getstarted.html)
