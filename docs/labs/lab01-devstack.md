# Lab 1: DevStack Installation

## Objective

Install a complete OpenStack environment on a single Ubuntu VM using DevStack.

---

## Prerequisites

| Requirement | Specification |
|-------------|---------------|
| OS | Ubuntu 22.04 LTS (Desktop or Server) |
| RAM | 8 GB minimum, 16 GB recommended |
| Disk | 40 GB free space |
| Network | Internet access |
| User | Non-root user with sudo privileges |

---

## Step 1: Prepare the System

!!! warning "Do NOT run as root"
    DevStack must be run as a regular user with sudo access.

```bash
# Update system packages
sudo apt update && sudo apt upgrade -y

# Install required packages
sudo apt install -y git net-tools

# Create a dedicated user (optional but recommended)
sudo useradd -s /bin/bash -d /opt/stack -m stack
echo "stack ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/stack
sudo -u stack -i
```

---

## Step 2: Clone DevStack

```bash
cd ~
git clone https://opendev.org/openstack/devstack
cd devstack
```

---

## Step 3: Configure DevStack

Create the `local.conf` file:

```bash
cat > local.conf << 'EOF'
[[local|localrc]]
ADMIN_PASSWORD=secret
DATABASE_PASSWORD=$ADMIN_PASSWORD
RABBIT_PASSWORD=$ADMIN_PASSWORD
SERVICE_PASSWORD=$ADMIN_PASSWORD
SERVICE_TOKEN=a682f596-76f3-11ef-9f11-0242ac120003

HOST_IP=$(hostname -I | awk '{print $1}')

# Services to enable
enable_service rabbit mysql key
enable_service n-api n-cpu n-cond n-sch n-novnc n-cauth
enable_service g-api g-reg
enable_service cinder c-sch c-api c-vol
enable_service neutron q-svc q-agt q-dhcp q-l3 q-meta

# Neutron networking
FLOATING_RANGE=192.168.56.0/24
FIXED_RANGE=10.0.0.0/24
FIXED_NETWORK_SIZE=256
FLAT_INTERFACE=eth1

# Logging
LOGFILE=$DEST/logs/stack.sh.log
LOGDAYS=2
EOF
```

!!! tip "Passwords"
    Change `ADMIN_PASSWORD` to something secure. All other passwords will use the same value for simplicity.

---

## Step 4: Run DevStack

```bash
./stack.sh
```

This will take **30–60 minutes** depending on your internet speed and hardware.

You should see output like:
```
=========================
DevStack Component Timing
=========================
Total runtime: 1842 seconds
... 
This is your host IP address: 192.168.56.101
This is your host IPv6 address: ::1
Horizon is now available at http://192.168.56.101/dashboard
```

---

## Step 5: Verify Installation

### Check Running Services

```bash
cd ~/devstack
source openrc admin

# List compute services
openstack compute service list

# List network agents
openstack network agent list

# List running VMs (should be empty)
openstack server list
```

### Access Horizon Dashboard

Open your browser and navigate to:

```
http://<YOUR_VM_IP>/dashboard
```

| Field | Value |
|-------|-------|
| Domain | `default` |
| User Name | `admin` |
| Password | `secret` (or your ADMIN_PASSWORD) |

---

## Step 6: Launch Your First VM (CLI)

```bash
# Source credentials
source ~/devstack/openrc demo

# Create a keypair
openstack keypair create --public-key ~/.ssh/id_rsa.pub mykey

# Launch an instance
openstack server create --image cirros-0.6.2-x86_64-disk \
  --flavor m1.tiny --network private \
  --key-name mykey my-first-vm

# Check status
openstack server list
```

---

## Common Issues

| Issue | Solution |
|-------|----------|
| `stack.sh` fails with apt errors | Run `sudo apt update` and retry |
| Out of memory | Increase VM RAM to 16GB |
| Network unreachable | Check `HOST_IP` in `local.conf` |
| Re-run after failure | `./unstack.sh && ./stack.sh` |

---

## Cleanup

```bash
cd ~/devstack
./unstack.sh      # Stop all services
./clean.sh        # Remove all DevStack data (DESTRUCTIVE)
```

---

## Submission

📋 **Lab Report Requirements:**

1. Screenshot of Horizon login page
2. Screenshot of Compute Services list
3. Screenshot of your running VM (`openstack server list`)
4. Answer: What does `HOST_IP` do in `local.conf`?
5. Answer: Why do we source `openrc` before running commands?

Submit via: [Google Classroom / LMS link]
