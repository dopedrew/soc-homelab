# Proxmox host configuration

- `interfaces.d-soclab` -> `/etc/network/interfaces.d/soclab`
  Creates the isolated `vmbr99` bridge (no physical uplink) and the scoped NAT
  rule that gives the lab outbound-only internet.

- `dnsmasq-soclab.conf` -> `/etc/dnsmasq.d/soclab.conf`
  DHCP + DNS for the lab, bound strictly to `vmbr99` and explicitly barred from
  answering on the production bridge.

See `docs/architecture.md` for the design rationale.
