# SOC Homelab

A self-contained Security Operations Center lab built on Proxmox: a Wazuh SIEM,
a Linux victim endpoint, a fully-tooled attacker box, and two Windows endpoints
(Server 2025 and Windows 11), all on an isolated, NAT-only network. Built to
practice detection engineering and incident response against real attacks, not
simulated ones.

## What this demonstrates

- **Network isolation done deliberately.** The lab runs intentionally
  vulnerable services and live offensive tooling, fully walled off from the
  production LAN with outbound-only internet. ([architecture](docs/architecture.md))
- **Mixed-OS endpoint monitoring.** Linux and Windows (server and client)
  endpoints enrolled against a single SIEM, which is what a real SOC monitors.
- **A proven detection pipeline.** A live SSH brute force, run from the attacker
  box, caught and compliance-classified by the SIEM in real time.
  ([detection demo](docs/detection-demo.md))
- **Real infrastructure debugging.** Getting Windows to install fully headless
  surfaced three non-obvious bugs (a filesystem limit, a driver mismatch, and
  an installer GUI fault) that were root-caused and worked around.
  ([write-up](docs/build-windows-vms.md))
- **Hands-on detection + remediation drills.** Five CVE walkthroughs, each
  standing up a real vulnerable service, exploiting it, finding the detection,
  then remediating and patching. ([exploit guides](exploit-guides/))

## The environment

| Host | Role | OS |
|------|------|-----|
| soc-siem | Wazuh SIEM (indexer + manager + dashboard) | Ubuntu 24.04 |
| soc-victim | Linux target endpoint (Wazuh agent) | Ubuntu 24.04 |
| soc-attacker | Offensive tooling (nmap, hydra, Metasploit, sqlmap, ...) | Ubuntu 24.04 |
| soc-dc01 | Windows server endpoint | Windows Server 2025 |
| soc-win11 | Windows client endpoint | Windows 11 Enterprise |

Isolated lab network `10.99.99.0/24` on a Proxmox bridge with no physical
uplink; outbound internet via a single scoped NAT rule.

```
   Production LAN  ----  vmbr0 (host uplink)  ----  Internet
                           |                          ^
                           |  (no L2 path)            | NAT, outbound only
                           |                          |
                        vmbr99  10.99.99.1/24  -------+   <-- isolated lab
                           |
     soc-siem   soc-victim   soc-attacker   soc-dc01   soc-win11
```

## Repository layout

```
docs/                     architecture, build write-ups, detection demo
config/proxmox/           isolated bridge + NAT, scoped DHCP (dnsmasq)
config/windows/           autounattend.xml answer files (passwords redacted)
exploit-guides/           five CVE detection + remediation walkthroughs
```

## Documentation

- [Architecture](docs/architecture.md) - isolation design, VM inventory, SIEM
- [Building the Linux VMs](docs/build-linux-vms.md) - cloud-image + cloud-init
- [Building the Windows VMs](docs/build-windows-vms.md) - headless unattend, and the three bugs
- [Detection demo](docs/detection-demo.md) - SSH brute force -> SIEM alert, end to end
- [Exploit guides](exploit-guides/) - Shellshock, Log4Shell, Apache path traversal, Spring4Shell, Struts2/OGNL

## A note on scope and safety

Everything here runs inside the isolated lab network and is for defensive
training: building detections, understanding attacker behavior, and practicing
remediation. The vulnerable targets are disposable containers on a network with
no path to anything real. Credentials shown in config files are placeholders;
real secrets are kept out of the repository.
