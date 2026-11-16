# SOC Homelab

A detection engineering lab you can build in a weekend: Wazuh SIEM, a Linux
victim, a fully-tooled attacker box, and two Windows endpoints, on a Proxmox
network with no physical uplink. Attack it for real, then go find the alert.

The lab is the starting point. The five CVE walkthroughs are the work: stand
up the vulnerable service, exploit it from the attacker box, find the
detection in the SIEM, then patch it and check the patch held.

```
   Production LAN  ----  vmbr0 (host uplink)  ----  Internet
                           |                          ^
                           |  (no L2 path)            | NAT, outbound only
                           |                          |
                        vmbr99  10.99.99.1/24  -------+   <-- isolated lab
                           |
     soc-siem   soc-victim   soc-attacker   soc-dc01   soc-win11
```

## What you need

- A Proxmox VE host with **16 vCPU, 24 GB RAM and 250 GB** free for all five
  VMs. Three VMs (SIEM, victim, attacker) run in about 8 vCPU / 14 GB if you
  skip Windows at first; the exploit guides only need those three.
- Evaluation ISOs for Windows Server 2025 and Windows 11 Enterprise, if you
  want the Windows half.
- Roughly a weekend. The Linux side is an afternoon; Windows unattended
  installs are the slow part, and the three bugs that cost the most time are
  already written up so you do not have to rediscover them.

## Build order

1. [Isolate the network first](docs/architecture.md). A Proxmox bridge with
   `bridge-ports none`, one scoped NAT rule, and dnsmasq bound to that bridge
   only. Do this before anything vulnerable exists.
2. [Build the Linux VMs](docs/build-linux-vms.md). Cloud images plus
   cloud-init: SIEM, victim, attacker.
3. [Build the Windows VMs](docs/build-windows-vms.md). Headless
   `autounattend.xml`, and the three non-obvious failures it surfaces.
4. [Prove the pipeline](docs/detection-demo.md). An SSH brute force from the
   attacker box, caught and compliance-classified by the SIEM in real time. If
   this does not fire, nothing after it is worth doing.
5. [Work the CVE guides](exploit-guides/). Detection and remediation drills,
   one CVE at a time.

## The environment

| Host | Role | OS | vCPU | RAM | Disk |
|------|------|-----|-----:|----:|-----:|
| soc-siem | Wazuh SIEM (indexer + manager + dashboard) | Ubuntu 24.04 | 4 | 8 GB | 60 GB |
| soc-victim | Linux target endpoint (Wazuh agent) | Ubuntu 24.04 | 2 | 2 GB | 20 GB |
| soc-attacker | Offensive tooling (nmap, hydra, Metasploit, sqlmap) | Ubuntu 24.04 | 2 | 4 GB | 30 GB |
| soc-dc01 | Windows server endpoint | Windows Server 2025 | 4 | 6 GB | 80 GB |
| soc-win11 | Windows client endpoint | Windows 11 Enterprise | 4 | 4 GB | 60 GB |

## The CVE walkthroughs

Each one stands up a real vulnerable service, exploits it, finds the detection,
then patches and verifies. All five are old and well documented, so the exploit
itself is easy to get working and the time goes on the detection.

| | CVE | What it teaches |
|---|---|---|
| 01 | [Shellshock](exploit-guides/01-shellshock-cve-2014-6271.md) ([CVE-2014-6271](https://nvd.nist.gov/vuln/detail/CVE-2014-6271)) | Command injection through a CGI environment variable |
| 02 | [Log4Shell](exploit-guides/02-log4shell-cve-2021-44228.md) ([CVE-2021-44228](https://nvd.nist.gov/vuln/detail/CVE-2021-44228)) | JNDI lookup leading to outbound callback, and why egress logging matters |
| 03 | [Apache path traversal](exploit-guides/03-apache-path-traversal-cve-2021-41773.md) ([CVE-2021-41773](https://nvd.nist.gov/vuln/detail/CVE-2021-41773)) | Path normalisation and what a traversal looks like in access logs |
| 04 | [Spring4Shell](exploit-guides/04-spring4shell-cve-2022-22965.md) ([CVE-2022-22965](https://nvd.nist.gov/vuln/detail/CVE-2022-22965)) | Class loader manipulation writing a web shell |
| 05 | [Struts2 OGNL](exploit-guides/05-struts2-ognl-cve-2017-5638.md) ([CVE-2017-5638](https://nvd.nist.gov/vuln/detail/CVE-2017-5638)) | Expression language injection in a malformed header |

## Repository layout

```
docs/                     architecture, build write-ups, detection demo
config/proxmox/           isolated bridge + NAT, scoped DHCP (dnsmasq)
config/windows/           autounattend.xml answer files (passwords redacted)
exploit-guides/           five CVE detection + remediation walkthroughs
```

## Contributing

A sixth CVE walkthrough is the most useful thing you can add, and
[CONTRIBUTING.md](CONTRIBUTING.md) has the shape to follow and the rules about
which CVEs belong here. Corrections to the build docs are just as welcome:
if a step broke on your Proxmox version, that is worth an issue.

## Scope and safety

Everything here runs inside the isolated lab network and exists for defensive
training: building detections, understanding attacker behaviour, and practising
response. The exploits target deliberately vulnerable services you stand up
yourself, on a network with no path back in. Do not point any of it at a system
you do not own.

The Windows answer files ship with `CHANGEME-LabAdminPassword` in every password
field. Set your own, and do not commit a real one back.

## Licence

MIT. See [LICENSE](LICENSE).
