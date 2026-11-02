# Detection demo: SSH brute force caught in real time

This is the end-to-end proof that the lab works as a detection environment:
a real attack run from the attacker box against the victim, caught and
classified by the SIEM, not a simulated or hand-written log entry.

## Setup

- **Target:** soc-victim, with SSH password authentication deliberately
  enabled (Ubuntu cloud images disable it by default; this is a lab, so it was
  turned back on to make the box attackable).
- **Agent:** soc-victim is enrolled as a Wazuh agent, so its `auth.log` flows
  to the manager on soc-siem.
- **Attacker:** soc-attacker, with `hydra` and a small wordlist.

## The attack

From soc-attacker, a reconnaissance scan followed by a credential-guessing run:

```bash
nmap -sV -T4 10.99.99.11                       # find the open SSH service
hydra -L users.txt -P passwords.txt -t 4 ssh://10.99.99.11
```

The real password was not in the wordlist, so every attempt failed, which is
exactly the point: failed authentication at volume is what a brute force looks
like, and what the SIEM should catch.

## The detection

On soc-siem, the alerts surfaced in real time in
`/var/ossec/logs/alerts/alerts.log`:

```
Rule: 5710 (level 5) -> 'sshd: Attempt to login using a non-existent user'
Src IP: 10.99.99.12
```

Each alert carried its full compliance mapping (PCI DSS, HIPAA, GDPR,
NIST 800-53, TSC) and the correct source IP of the attacker box. The brute
force was visible as a burst of failed-auth events from a single source,
which is the signal a SOC analyst triages.

## Why this matters

The value is not the brute force itself, it is the proven pipeline:
attack -> host log -> agent -> manager -> classified, compliance-tagged alert.
Once that path is confirmed, the same environment can exercise any detection:
the CVE exploit walkthroughs in `exploit-guides/` are built to run the same
loop (stand up a vulnerable service, attack it from soc-attacker, find the
detection on soc-siem, then remediate and patch).
