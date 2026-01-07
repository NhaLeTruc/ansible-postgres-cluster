# Infrastructures buid then test with mostly Terraform and Ansible

## Requirements

Minimum supported Ansible version: 9.0.0 (ansible-core 2.16.0)

This playbook requires root privileges or sudo.
Ansible ([What is Ansible](https://www.ansible.com/how-ansible-works/)?)
if dcs_type: "consul", please install consul role requirements on the control node:
`ansible-galaxy install -r roles/consul/requirements.yml`

### Port requirements

List of required TCP ports that must be open for the database cluster:

- `5432` (postgresql)
- `6432` (pgbouncer)
- `8008` (patroni rest api)
- `2379`, `2380` (etcd)
for the scheme "[Type A] PostgreSQL High-Availability with Load Balancing":
- `5000` (haproxy - (read/write) master)
- `5001` (haproxy - (read only) all replicas)
- `5002` (haproxy - (read only) synchronous replica only)
- `5003` (haproxy - (read only) asynchronous replicas only)
- `7000` (optional, haproxy stats)
for the scheme "[Type C] PostgreSQL High-Availability with Consul Service Discovery (DNS)":
- `8300` (Consul Server RPC)
- `8301` (Consul Serf LAN)
- `8302` (Consul Serf WAN)
- `8500` (Consul HTTP API)
- `8600` (Consul DNS server)

## Recommenations

- **linux (Operation System)**:
Update your operating system on your target servers before deploying;
Make sure you have time synchronization is configured (NTP).
Specify `ntp_enabled:'true'` and `ntp_servers` if you want to install and configure the ntp service.
- **DCS (Distributed Consensus Store)**:
Fast drives and a reliable network are the most important factors for the performance and stability of an etcd (or consul) cluster.
Avoid storing etcd (or consul) data on the same drive along with other processes (such as the database) that are intensively using the resources of the disk subsystem!
Store the etcd and postgresql data on **different** disks (see `etcd_data_dir`, `consul_data_path` variables), use ssd drives if possible.
See [hardware recommendations](https://etcd.io/docs/v3.3/op-guide/hardware/) and [tuning](https://etcd.io/docs/v3.3/tuning/) guides.
It is recommended to deploy the DCS cluster on dedicated servers, separate from the database servers.
- **Placement of cluster members in different data centers**:
If you’d prefer a cross-data center setup, where the replicating databases are located in different data centers, etcd member placement becomes critical.
There are quite a lot of things to consider if you want to create a really robust etcd cluster, but there is one rule: *do not placing all etcd members in your primary data center*. See some [examples](https://www.cybertec-postgresql.com/en/introduction-and-how-to-etcd-clusters-for-patroni/).
- **How to prevent data loss in case of autofailover (synchronous_modes)**:
Due to performance reasons, a synchronous replication is disabled by default.
To minimize the risk of losing data on autofailover, you can configure settings in the following way:
- synchronous_mode: 'true'
- synchronous_mode_strict: 'true'
- synchronous_commit: 'on' (or 'remote_apply')

## Getting Started with Homelab

Tested on 24th April 2025 Proxmox server.

## Install requirements

```bash
ansible-galaxy install -r requirements.yml
```

Optionally generate a self-signed certificate, private key for TLS encryption of Vault in /certs/ directory:

```bash
openssl req -newkey rsa:2048 -keyout ansible_key.pem -out ansible.csr

openssl x509 -signkey ansible_key.pem -in ansible.csr -req -days 365 -out ansible.crt

chmod 600 ansible_key.pem ansible.crt ansible.csr
```

## Create VM manually on PROXMOX shell

```bash

wget https://cloud.debian.org/images/cloud/bullseye/20250303-2040/debian-11-generic-amd64-20250303-2040.qcow2

qm create 902 \
    --name "debian-11-amd64" \
    --net0 "virtio,bridge=vmbr0" \
    --serial0 socket \
    --vga serial0 \
    --scsihw virtio-scsi-pci \
    --scsi0 "local:0,import-from=/root/debian-11-generic-amd64-20250303-2040.qcow2" \
    --bootdisk scsi0 \
    --boot "order=scsi0" \
    --ide1 "local-lvm:cloudinit" \
    --ipconfig0 "ip=dhcp" \
    --ciuser admin\
    --cipassword debian\
    --ostype l26 \
    --cores 2 \
    --sockets 2 \
    --memory 4096 \
    --agent 1

qm resize 902 scsi0 10G

qm template 902

```

## Filling in variables for packers and terraform

Run **/bin/generate-vars** after installing **python-hcl2**. This would create variable files in designated directories. Make sure to add them into *.gitignore*.

Packer will ssh into the VM above clone and do effectively these tasks

```bash

# Configure Cloud-Init/IP Config & User/Password
# Login and install qemu-guest-agent

sudo apt update && apt -y install qemu-guest-agent

systemctl enable qemu-guest-agent
systemctl start qemu-guest-agent

systemctl status qemu-guest-agent

```

## Run Packer to build base image

Proxmox couldn't create golden image template yet at the moment. So Packer will clone the manually created template then configurate it instead.

Crucially **qemu-guest-agent** will be installed into the golden image template. This allow terraform to utilize it in create the cluster's nodes.

## Check Proxmox VM config info

```bash
pvesh get /nodes/pve/qemu/$VMID/config --output-format json-pretty

terraform providers

terraform -upgrade
```

## Packer have been moved to separated repository

Terraform is responsible for creating the vm needed for the cluster.

Ansible handles the heavy lifting of setting up and configurating Postgres HA cluster.

## Terraform IaC

Packer template has ssh key installed in them. Specifies ssh access in terraform code created issue of ssh acess sometimes.

Be mindful of choosing static ips for VM. Some local ips could have been taken by other service/servers.

```hcl
clone {
  datastore_id = var.disk_datastore
  vm_id        = var.clone_template_id
  retries      = 3
  full         = true
}
```

Clone `full = true` is very important. Keep this option on.

## Add files for ansible-core 2.14.18 to work instead of 2.16+

Some code needed to be added into project for ansible to work.

+ `.\ansible\module_utils\common\file.py`
+ `.\ansible\module_utils\urls.py`

## proxmox command stop/start multiple vms

```bash
# First time only
apt-get update && apt-get install parallel

# Generate file with all related VM IDs
seq 200 210 >> vm_list.txt

# Stop all VMs
cat vm_list.txt | parallel -j 11 qm stop {}

# Start all VMs
cat vm_list.txt | parallel -j 11 qm start {}

# Remove all VMs whose ID is in vm_list.txt
cat vm_list.txt | parallel -j 11 qm destroy {}
```

## Resize Proxmox VM disk size

To increase the size of a virtual machine's disk in Proxmox, you need to resize the disk within the Proxmox GUI and then expand the partition and filesystem inside the guest operating system. First, shut down the VM and resize the disk using the Proxmox web interface, then start the VM and use tools within the guest OS to expand the partition and filesystem to utilize the newly allocated space.

Steps to resize a Proxmox VM disk:

1. Shut down the VM: Before resizing, ensure the virtual machine is powered off.
2. Navigate to the VM's hardware settings: In the Proxmox web interface, select the VM, go to the "Hardware" tab, and choose the hard disk you want to resize.
3. Resize the disk: Click on "Disk Actions" and then "Resize".
4. Enter the new size: Specify the desired new size for the disk.
5. Start the VM: Power on the virtual machine.
6. Expand the partition and filesystem (within the guest OS): Use tools like gparted, parted, or commands specific to your guest OS (e.g., resize2fs for ext4, xfs_growfs for xfs) to expand the partition and filesystem to use the newly allocated disk space.
   1. backup the system
   2. disable swap
   3. remove swap
   4. remove extended partition
   5. resize /dev/sda1 (disk size - wanted swap size)
   6. create new swap partition /dev/sda2 or on extended
   7. resize2fs /dev/sda1
   8. mkswap on swap partition
   9. add swap partition to /etc/fstab
7. Verify the change: Check the storage settings within the guest OS to confirm the increased disk size.

> This is the last resort, not reccommended. Scale horizontally!

```bash
# View disk infos
lsblk
df -h

# Install disk management tool
sudo apt install lvm2

```

> If your VM uses LVM or ZFS, you'll need to use specific commands (e.g., lvresize for LVM) to expand the logical volume or ZFS volume.
> Always back up your VM before resizing the disk, especially if you are working with LVM or ZFS. 
> How you expand the partition and filesystem within the guest OS depends on the guest OS and its partitioning scheme (e.g., LVM, ZFS, separate /home partition).

Researches here:

1. [Add local module](https://docs.ansible.com/ansible/latest/dev_guide/developing_locally.html#adding-a-module-or-plugin-outside-of-a-collection)
2. [Add local module_utils](https://docs.ansible.com/ansible/latest/dev_guide/developing_module_utilities.html)
3. [deb822_repository module](https://github.com/ansible/ansible/blob/devel/lib/ansible/modules/deb822_repository.py)
4. [urls module_utils](https://github.com/ansible/ansible/blob/devel/lib/ansible/module_utils/urls.py)
