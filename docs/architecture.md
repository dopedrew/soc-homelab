# Architecture

## Design goals

1. **Fully isolated from the production network.** The lab runs deliberately
   vulnerable services and live attack tooling. Nothing in it can reach, or be
   reached from, the real LAN.
2. **Outbound internet for package installs only.** VMs need to pull packages
   and updates, but nothing outside the lab should ever be able to initiate a
   connection into it.
3. **Realistic, mixed-OS environment.** A SIEM, a Linux victim, an attacker
   box, and two Windows endpoints (Server and client), so detections can be
   exercised across the operating systems a real SOC actually monitors.

## Network isolation

The lab lives on its own Proxmox Linux bridge, `vmbr99`, with **no physical
NIC attached** (`bridge-ports none`). With no uplink port, there is no layer-2
path from the lab to the production bridge (`vmbr0`) or anything behind it.

Outbound internet is provided by a single NAT rule on the Proxmox host,
scoped to the lab subnet only:

```
iptables -t nat -A POSTROUTING -s 10.99.99.0/24 -o vmbr0 -j MASQUERADE
```

Because this is source-NAT on an otherwise unrouted bridge, traffic can leave
the lab (for `apt`/Windows Update) but nothing can route back in unsolicited.
A `dnsmasq` instance bound strictly to `vmbr99` hands out DHCP and DNS inside
the lab and is explicitly prevented from answering on the production bridge.

See `config/proxmox/interfaces.d-soclab` and
`config/proxmox/dnsmasq-soclab.conf` for the exact configuration.

```
                        Internet
                           ^
                           | NAT (MASQUERADE, outbound only)
                           |
   Production LAN  ----  vmbr0 (Proxmox host uplink)
   10.10.0.0/24           |
                          |   (no bridge-ports: no L2 path across)
                          |
                       vmbr99   10.99.99.1/24   <-- isolated lab bridge
                          |
        +--------+--------+--------+--------+--------+
        |        |        |        |        |        |
     soc-siem  victim  attacker  soc-dc01  soc-win11
```

## VM inventory

| Host | Role | OS | vCPU | RAM | Disk | NIC |
|------|------|-----|------|-----|------|-----|
| soc-siem | SIEM / log aggregation | Ubuntu 24.04 | 4 | 8 GB | 60 GB | virtio |
| soc-victim | Linux target endpoint | Ubuntu 24.04 | 2 | 2 GB | 20 GB | virtio |
| soc-attacker | Offensive tooling | Ubuntu 24.04 | 2 | 4 GB | 30 GB | virtio |
| soc-dc01 | Windows server endpoint | Windows Server 2025 Eval | 4 | 6 GB | 80 GB | e1000 |
| soc-win11 | Windows client endpoint | Windows 11 Enterprise Eval | 4 | 4 GB | 60 GB | e1000 |

Addresses are handed out by DHCP from `10.99.99.50-99`; the Proxmox host is
the gateway at `10.99.99.1`.

## Provisioning approach

- **Linux VMs** are built from Ubuntu cloud images with `cloud-init` for
  fully unattended, hands-off provisioning (user, SSH key, static networking).
- **Windows VMs** use `autounattend.xml` answer files for unattended install.
  Getting these to install truly headlessly surfaced two real bugs worth
  reading: see `build-windows-vms.md`.

## The SIEM

The SIEM host runs Wazuh (indexer + manager + dashboard, single-node
all-in-one). Endpoints are enrolled as Wazuh agents so host logs, file
integrity events, and authentication activity flow back for detection and
alerting. Wazuh ships rules pre-mapped to PCI DSS, HIPAA, GDPR, and
NIST 800-53, so a single detected event carries its compliance context.

A worked end-to-end detection (an SSH brute force caught in real time) is in
`detection-demo.md`.
