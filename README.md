# Build a Kubernetes cluster using K3s via Ansible

Author: <https://github.com/itwars>  
Current Maintainer: <https://github.com/dereknola>

Easily bring up a cluster on machines running:

- [X] Debian
- [X] Ubuntu
- [X] Raspberry Pi OS
- [X] RHEL Family (CentOS, Redhat, Rocky Linux...)
- [X] SUSE Family (SLES, OpenSUSE Leap, Tumbleweed...)
- [X] ArchLinux

on processor architectures:

- [X] x64
- [X] arm64
- [X] armhf

## System requirements

The control node must have Ansible 2.10.0+

All managed nodes in inventory must have:
- Passwordless SSH access
- Root access (or a user with equivalent permissions) 

It is also recommended that all managed nodes disable firewalls and swap. See [K3s Requirements](https://docs.k3s.io/installation/requirements) for more information.

## Usage

First copy the sample inventory to `inventory.yml`.

```bash
cp inventory-sample.yml inventory.yml
```

Second edit the inventory file to match your cluster setup. For example:
```bash
k3s_cluster:
  children:
    server:
      hosts:
        192.16.35.11:
    agent:
      hosts:
        192.16.35.12:
        192.16.35.13:
```

If needed, you can also edit `vars` section at the bottom to match your environment.

If multiple hosts are in the server group the playbook will automatically setup k3s in HA mode with embedded etcd.
An odd number of server nodes is required (3,5,7). Read the [official documentation](https://docs.k3s.io/datastore/ha-embedded) for more information.

Setting up a loadbalancer or VIP beforehand to use as the API endpoint is possible but not covered here.


Start provisioning of the cluster using the following command:

```bash
ansible-playbook playbook/site.yml -i inventory.yml
```

## Upgrading

A playbook is provided to upgrade K3s on all nodes in the cluster. To use it, update `k3s_version` with the desired version in `inventory.yml` and run:

```bash
ansible-playbook playbook/upgrade.yml -i inventory.yml
```

## Airgap Install

Airgap installation is supported via the `airgap_dir` variable. This variable should be set to the path of a directory containing the K3s binary and images. The release artifacts can be downloaded from the [K3s Releases](https://github.com/k3s-io/k3s/releases). You must download the appropriate images for you architecture (any of the compression formats will work).

An example folder for an x86_64 cluster:
```bash
$ ls ./playbook/my-airgap/
total 248M
-rwxr-xr-x 1 $USER $USER  58M Nov 14 11:28 k3s
-rw-r--r-- 1 $USER $USER 190M Nov 14 11:30 k3s-airgap-images-amd64.tar.gz

$ cat inventory.yml
...
airgap_dir: ./my-airgap # Paths are relative to the playbook directory
```

Additionally, if deploying on a OS with SELinux, you will also need to download the latest [k3s-selinux RPM](https://github.com/k3s-io/k3s-selinux/releases/latest) and place it in the airgap folder.


It is assumed that the control node has access to the internet. The playbook will automatically download the k3s install script on the control node, and then distribute all three artifacts to the managed nodes. 

## Kubeconfig

After successful bringup, the kubeconfig of the cluster is copied to the control node  and set as default (`~/.kube/config`).
Assuming you have [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl) installed, you to confirm access to your **Kubernetes** cluster use the following:

```bash
kubectl get nodes
```

## OS Upgrade

A playbook is provided to perform a rolling OS upgrade (`apt dist-upgrade`) across all nodes in the cluster, one node at a time.

```bash
ansible-playbook playbook/os-upgrade.yml -i inventory.yml
```

To upgrade only agents or only the server:

```bash
ansible-playbook playbook/os-upgrade.yml -i inventory.yml --limit agent
ansible-playbook playbook/os-upgrade.yml -i inventory.yml --limit server
```

**How it works:**

- Each agent node is processed one at a time (`serial: 1`):
  1. `kubectl cordon` + `kubectl drain` to evict workloads
  2. `apt-get dist-upgrade` — upgrades all packages including kernel
  3. Reboot if `/var/run/reboot-required` exists (timeout: 600s)
  4. `kubectl uncordon` + 60s pause for pods to reschedule
- The server (control plane) is upgraded last, without drain (single control plane). A brief API unavailability (~2–3 min) is expected during its reboot.

### Raspberry Pi nodes booting from a NAS (NFS + iSCSI)

This playbook handles a specific architecture where Raspberry Pi nodes have no local storage: the boot partition is served over NFS and the root filesystem is mounted over iSCSI from a Synology NAS.

**Architecture overview:**

```
┌─────────────────────────────────────────────────────┐
│  Synology NAS (192.168.1.10)                        │
│                                                     │
│  /volume1/rpi-tftpboot/<serial>/   ← NFS boot dir  │
│    kernel8.img                     ← kernel        │
│    initramfs8                      ← initramfs     │
│    config.txt, cmdline.txt, ...                     │
│                                                     │
│  iSCSI target per node             ← root (/)       │
└─────────────────────────────────────────────────────┘
         │ NFS (/boot)          │ iSCSI (/)
         ▼                      ▼
┌────────────────────────────────────────────────────┐
│  Raspberry Pi node                                 │
│   /boot        ← NFS-mounted boot dir              │
│   /            ← iSCSI root filesystem             │
│   /boot/firmware ← bind mount of /boot (see below) │
└────────────────────────────────────────────────────┘
```

Each Pi has its own dedicated directory on the NAS (named after its serial number). The NAS serves `kernel8.img` and `initramfs8` via TFTP/NFS at boot time. The root filesystem is a per-node iSCSI LUN on the NAS.

**The `/boot/firmware` bind mount:**

Modern Raspberry Pi OS kernel and firmware packages (bookworm and later) write their files to `/boot/firmware/`, but on this network-boot setup the boot partition is mounted at `/boot/`. Without intervention, kernel post-install scripts fail with:

```
raspi-firmware: missing /boot/firmware, did you forget to mount it?
```

The playbook solves this by creating a bind mount so both paths point to the same NFS share:

```
/boot  →  bind-mounted at  →  /boot/firmware
```

This is set up in `/etc/fstab` on each node:

```
192.168.1.10:/volume1/rpi-tftpboot/<serial>  /boot           nfs   defaults,_netdev  0 0
/boot                                         /boot/firmware  none  bind,_netdev      0 0
```

When `apt` installs a new kernel, the post-install scripts write `kernel8.img` and `initramfs8` to `/boot/firmware/`, which resolves to `/boot/` and is therefore written directly to the NFS share on the NAS. The Pi picks up the new files on next boot.

**iSCSI in the initramfs:**

For the Pi to mount its root filesystem on boot, the iSCSI initiator must be present in the initramfs. This is handled automatically by `update-initramfs` because `/etc/iscsi/iscsi.initramfs` exists on each node (its presence is the trigger for the open-iscsi initramfs hook).

The `config.txt` on each node contains `auto_initramfs=1`, which instructs the Pi firmware to automatically load the initramfs matching the kernel (`kernel8.img` → `initramfs8`).

You can verify iSCSI is included in the current initramfs with:

```bash
lsinitramfs /boot/initramfs8 | grep iscsi
```

### Ongoing security maintenance

A role is provided to configure `unattended-upgrades` for automatic security-only patches (no reboot — pair with [kured](https://github.com/kubereboot/kured) for automated rolling reboots):

```bash
ansible-playbook playbook/site.yml -i inventory.yml --tags os-maintenance
```

Or apply the role directly:

```yaml
- hosts: k3s_cluster
  become: true
  roles:
    - role: os-maintenance
```

## Local Testing

A Vagrantfile is provided that provision a 5 nodes cluster using Vagrant (LibVirt or Virtualbox as provider). To use it:

```bash
vagrant up
```

By default, each node is given 2 cores and 2GB of RAM and runs Ubuntu 20.04. You can customize these settings by editing the `Vagrantfile`.

## Need More Features?

This project is intended to provide a "vanilla" K3s install. If you need more features, such as:
- Private Registry
- Advanced Storage (Longhorn, Ceph, etc)
- External Database
- External Load Balancer or VIP
- Alternative CNIs

See these other projects:
- https://github.com/PyratLabs/ansible-role-k3s
- https://github.com/techno-tim/k3s-ansible
- https://github.com/jon-stumpf/k3s-ansible
- https://github.com/alexellis/k3sup
