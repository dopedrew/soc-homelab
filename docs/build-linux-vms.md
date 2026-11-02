# Building the Linux VMs

All three Linux hosts (SIEM, victim, attacker) are built the same way: an
Ubuntu cloud image plus `cloud-init`. This is fully hands-off, there is no
interactive installer to click through, which matters when the whole lab is
driven remotely.

## Why cloud images, not the ISO installer

The Ubuntu live-server ISO expects an interactive install. In a headless,
scripted build that is a dead end. Ubuntu's cloud images are pre-installed
disk images designed to be customized at first boot by `cloud-init`, so the
VM comes up already configured: user created, SSH key installed, networking
set, no keyboard required.

## Outline

1. Download the Ubuntu 24.04 cloud image (`.img`) to the Proxmox host.
2. Create the VM and import the cloud image as its disk.
3. Attach a `cloud-init` drive and set user, SSH key, and IP config.
4. Boot. `cloud-init` applies everything on first start.

```bash
# on the Proxmox host (VMID and sizing per docs/architecture.md)
qm create 103 --name soc-siem --memory 8192 --cores 4 \
  --net0 virtio,bridge=vmbr99 --scsihw virtio-scsi-pci --ostype l26
qm importdisk 103 noble-server-cloudimg-amd64.img local-lvm
qm set 103 --scsi0 local-lvm:vm-103-disk-0
qm set 103 --ide2 local-lvm:cloudinit
qm set 103 --boot order=scsi0 --serial0 socket --vga serial0
qm set 103 --ciuser dopedrew --sshkeys ~/.ssh/id_ed25519.pub
qm set 103 --ipconfig0 ip=10.99.99.10/24,gw=10.99.99.1
qm resize 103 scsi0 60G
qm start 103
```

## SIEM

On soc-siem, Wazuh is installed with the official single-node assistant:

```bash
curl -sO https://packages.wazuh.com/4.9/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

This installs the indexer, manager, Filebeat, and dashboard in one pass and
prints the generated admin credentials at the end. Those credentials are kept
out of this repo; store them in a password manager, not in the tree.

## Attacker toolkit

soc-attacker carries a standard offensive toolkit: `nmap`, `hydra`, `sqlmap`,
`nikto`, `netcat`, `impacket`, `dirb`, `whatweb`, Metasploit, plus SecLists
and Responder cloned into `/opt`. These are used only against lab targets on
the isolated network.

## A real gotcha: SSH command timeouts and racing qm calls

When driving `qm set`/`qm resize` over SSH with a client-side `timeout`, if the
local timeout fires while the remote command chain is still running, the SSH
client connection dies but the **remote** shell keeps executing. Kick off the
same chain again and you get two near-simultaneous `qm set ... -scsi0` runs
against the same VM config, which corrupted the LVM volume twice
("no such logical volume") before it was diagnosed from `journalctl` showing
the duplicate entries. The fix: generous timeouts, and verify no stray `qm`
processes remain before any retry.
