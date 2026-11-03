# Lab 2: Horizon Dashboard & Project Management

## Objective

Navigate the Horizon web interface and perform basic cloud operations through the GUI.

---

## Part 1: Horizon Overview

### Login

1. Open browser: `http://<YOUR_DEVSTACK_IP>/dashboard`
2. Domain: `default`
3. Username: `admin`
4. Password: your `ADMIN_PASSWORD`

### Dashboard Layout

```
┌──────────────────────────────────────────────────────┐
│  Project | Admin | Identity | Settings               │
├──────────────────┬───────────────────────────────────┤
│                  │                                   │
│  Overview        │   ╔═══════════╗                  │
│  Instances       │   ║ Instances ║  0               │
│  Volumes         │   ╚═══════════╝                  │
│  Networks        │                                   │
│  Images          │   ╔═══════════╗                  │
│  Security Groups │   ║  Volumes  ║  0               │
│  Network Topology│   ╚═══════════╝                  │
│                  │                                   │
└──────────────────┴───────────────────────────────────┘
```

---

## Part 2: Create a Project and User

### Via Horizon

1. Navigate to **Identity → Projects → Create Project**
2. Fill in:
   - Project Name: `student-lab`
   - Description: `Lab project for OpenStack course`
3. Click **Create Project**

### Create a User

1. Navigate to **Identity → Users → Create User**
2. Fill in:
   - User Name: `student01`
   - Password: `password123`
   - Primary Project: `student-lab`
   - Role: `member`
3. Click **Create User**

---

## Part 3: Launch an Instance via Horizon

### Step 1: Upload an Image

1. Go to **Project → Compute → Images → Create Image**
2. Choose image source: URL
3. Image Location: `https://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img`
4. Image Format: QCOW2 — QEMU Emulator
5. Click **Create Image**

### Step 2: Create a Network

1. Go to **Project → Network → Networks → Create Network**
2. Network Name: `student-net`
3. Subnet Name: `student-subnet`
4. Network Address: `192.168.100.0/24`
5. Gateway IP: `192.168.100.1`
6. Click **Create**

### Step 3: Create a Router

1. **Project → Network → Routers → Create Router**
2. Router Name: `student-router`
3. External Network: `public`
4. Click **Create Router**
5. Click router name → **Interfaces → Add Interface**
6. Subnet: `student-subnet`
7. Click **Submit**

### Step 4: Configure Security Group

1. **Project → Network → Security Groups**
2. Click **Manage Rules** on `default`
3. **Add Rule**:
   - Rule: SSH
   - Remote: CIDR
   - CIDR: `0.0.0.0/0`
4. Add another:
   - Rule: ICMP (for ping)

### Step 5: Launch Instance

1. **Project → Compute → Instances → Launch Instance**
2. Details:
   - Instance Name: `vm-01`
3. Source:
   - Select Boot Source: Image
   - Select the cirros image
4. Flavor: `m1.tiny`
5. Networks: `student-net`
6. Security Groups: `default`
7. Key Pair: Create or import one
8. Click **Launch Instance**

---

## Part 4: Allocate and Associate a Floating IP

1. **Project → Network → Floating IPs → Allocate IP**
2. Pool: `public`
3. Click **Allocate IP**
4. Click **Associate** → Select `vm-01`
5. Click **Associate**

!!! success "Test Connectivity"
    From your host machine:
    ```bash
    ping <FLOATING_IP>
    ssh cirros@<FLOATING_IP>
    # Password: gocubsgo
    ```

---

## Part 5: Create a Volume and Attach

1. **Project → Volumes → Volumes → Create Volume**
   - Volume Name: `data-vol`
   - Size: 2 GB
2. Click **Create Volume**
3. Click **Manage Attachments** → Attach to `vm-01`
4. SSH into the VM and verify:
   ```bash
   ssh cirros@<FLOATING_IP>
   lsblk
   sudo mkfs.ext4 /dev/vdb
   sudo mkdir /mnt/data
   sudo mount /dev/vdb /mnt/data
   df -h
   ```

---

## Submission

📋 **Lab Report Requirements:**

1. Screenshot of the Project Overview page
2. Screenshot showing the created `student-net` network topology
3. Screenshot of `vm-01` running with a floating IP
4. Screenshot of the SSH session inside the VM
5. Answer: What is the difference between a private network and a public (external) network?
6. Answer: Why do we need a router between them?
