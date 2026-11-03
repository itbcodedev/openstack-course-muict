# Lab 3: OpenStack CLI Operations

## Objective

Master the OpenStack command-line client for managing cloud resources programmatically.

---

## Prerequisites

```bash
# Source your DevStack credentials
source ~/devstack/openrc admin

# Verify CLI is working
openstack --version
```

---

## Part 1: CLI Basics

### Command Structure

```bash
openstack <service> <action> <resource> [options]
```

Examples:
```bash
openstack server list              # List VMs
openstack network list             # List networks
openstack flavor list              # List flavors
openstack image list               # List images
```

### Get Help

```bash
openstack --help                   # Top-level help
openstack server --help            # Server subcommand help
openstack server create --help     # Specific command help
```

---

## Part 2: Identity & Project Management

### Create a Project

```bash
openstack project create \
  --description "MUICT OpenStack Course" \
  --enable \
  muict-course
```

### Create a User

```bash
openstack user create \
  --password-prompt \
  --email student@muict.ac.th \
  --project muict-course \
  --enable \
  student01
```

### Assign a Role

```bash
openstack role add \
  --user student01 \
  --project muict-course \
  member

# Verify
openstack role assignment list --user student01 --project muict-course
```

### Create a Service Account

```bash
openstack user create \
  --password servicepass \
  --project service \
  --enable \
  heat-domain-admin
```

---

## Part 3: Compute Operations

### Flavors

```bash
# List all flavors
openstack flavor list

# Create a custom flavor
openstack flavor create \
  --ram 2048 \
  --disk 20 \
  --vcpus 2 \
  --public \
  m1.student
```

### Key Pairs

```bash
# Generate SSH key
ssh-keygen -t rsa -f ~/.ssh/openstack_key -N ""

# Upload to OpenStack
openstack keypair create --public-key ~/.ssh/openstack_key.pub openstack_key

# List keypairs
openstack keypair list
```

### Launch an Instance

```bash
# Find available images and networks
openstack image list
openstack network list

# Launch
openstack server create \
  --image cirros-0.6.2-x86_64-disk \
  --flavor m1.tiny \
  --network private \
  --key-name openstack_key \
  --security-group default \
  --wait \
  my-cli-vm

# Check status
openstack server show my-cli-vm
openstack server list
```

### Instance Lifecycle

```bash
# Stop the instance
openstack server stop my-cli-vm

# Start the instance
openstack server start my-cli-vm

# Reboot (soft)
openstack server reboot my-cli-vm

# Reboot (hard)
openstack server reboot --hard my-cli-vm

# Suspend / Resume
openstack server suspend my-cli-vm
openstack server resume my-cli-vm

# Delete
openstack server delete my-cli-vm
```

### Console Access

```bash
# Get VNC console URL
openstack console url show --vnc my-cli-vm
```

---

## Part 4: Network Operations

### Create Network and Subnet

```bash
# Create network
openstack network create student-cli-net

# Create subnet
openstack subnet create \
  --network student-cli-net \
  --subnet-range 10.100.0.0/24 \
  --gateway 10.100.0.1 \
  --allocation-pool start=10.100.0.10,end=10.100.0.250 \
  --dns-nameserver 8.8.8.8 \
  student-cli-subnet
```

### Create Router

```bash
# Create router
openstack router create student-cli-router

# Connect to external network
openstack router set --external-gateway public student-cli-router

# Connect to internal subnet
openstack router add subnet student-cli-router student-cli-subnet

# Verify
openstack router show student-cli-router
```

### Security Groups

```bash
# Create a new security group
openstack security group create ssh-access

# Add SSH rule
openstack security group rule create \
  --protocol tcp \
  --dst-port 22 \
  --remote-ip 0.0.0.0/0 \
  ssh-access

# Add ICMP (ping)
openstack security group rule create \
  --protocol icmp \
  --remote-ip 0.0.0.0/0 \
  ssh-access

# Apply to instance
openstack server add security group my-cli-vm ssh-access
```

### Floating IPs

```bash
# Allocate floating IP
openstack floating ip create public

# Get the allocated IP
FLOATING_IP=$(openstack floating ip list -f value -c 'Floating IP Address' | head -1)
echo $FLOATING_IP

# Associate with instance
openstack server add floating ip my-cli-vm $FLOATING_IP

# Verify
openstack server show my-cli-vm -c addresses
```

---

## Part 5: Volume Operations

```bash
# Create volume
openstack volume create --size 5 --description "Data disk" data-volume

# Attach to instance
openstack server volume attach my-cli-vm data-volume

# Detach
openstack server volume detach my-cli-vm data-volume

# Delete
openstack volume delete data-volume
```

---

## Part 6: Scripting with CLI

### Batch VM Creation

```bash
#!/bin/bash
# batch_create.sh

NETWORK_ID=$(openstack network show private -f value -c id)
IMAGE_ID=$(openstack image show cirros-0.6.2-x86_64-disk -f value -c id)

for i in {1..3}; do
  openstack server create \
    --image $IMAGE_ID \
    --flavor m1.tiny \
    --network $NETWORK_ID \
    --key-name openstack_key \
    batch-vm-$i
done

# Wait for all to be active
for i in {1..3}; do
  openstack server wait batch-vm-$i
done

openstack server list
```

---

## Submission

📋 **Lab Report Requirements:**

1. Output of `openstack server list` after creating instances
2. Screenshot or output showing the floating IP association
3. Script output from the batch VM creation
4. Answer: What is the difference between `openstack server stop` and `openstack server suspend`?
5. Answer: Why is it useful to use `--wait` in automation scripts?
