# CLAUDE.md

Ansible playbooks that provision and maintain the k3s home cluster: 6 Raspberry Pi
(`k3s-master` 192.168.1.100, agents 192.168.1.101–105) booting over the network from the
Synology NAS 192.168.1.10 (`/boot` on NFS, `/` on one iSCSI LUN per node, k3s datastore
in Postgres on the NAS). Workloads live in `seblaporte/k3s-at-home` (Flux).

Fork of `k3s-io/k3s-ansible`. Default branch: `master`. CI: yamllint + ansible-lint.

## Layout

- `playbook/site.yml` installs k3s, `upgrade.yml` upgrades it (`k3s_version`),
  `os-upgrade.yml` upgrades the OS node by node, `reboot.yml`, `reset.yml`.
- `roles/`: `prereq`, `raspberrypi`, `k3s_server`, `k3s_agent`, `k3s_upgrade`,
  `os-maintenance` (unattended-upgrades), `airgap`.
- `inventory.yml` is gitignored and **contains secrets** (k3s token, Postgres datastore
  URL): never read or print it. `inventory-sample.yml` shows its structure.

## Rules

- Run Ansible from the repo root with `-i inventory.yml`.
- Against the real nodes, run `--check --diff` first. Applying a playbook, rebooting or
  stopping k3s on a node needs the user's explicit go-ahead in the session.
- Never bring eth0 down on a node (`nmcli connection down`, `ifdown`…): the root
  filesystem is on iSCSI and the node freezes immediately.
- A node must never lose the NAS while running. NAS maintenance (DSM update, reboot,
  power cut) follows the `nas-maintenance` skill of the hub repo `seblaporte/homelab`.
- The DSM firewall only opens TFTP/NFS/iSCSI/Postgres to 192.168.1.100–105: a new node
  outside this range cannot boot.
