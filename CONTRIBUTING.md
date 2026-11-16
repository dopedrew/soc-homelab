# Contributing

The most useful contribution is **another CVE walkthrough**. Everything else in
this repo exists to make those possible.

## Adding a CVE walkthrough

Copy the shape of
[`01-shellshock-cve-2014-6271.md`](exploit-guides/01-shellshock-cve-2014-6271.md)
and keep its six sections, in this order:

| Section | What belongs in it |
|---|---|
| The vulnerability | What is actually broken, in a paragraph. No CVSS recitation. |
| Lab setup | Exact commands to stand the vulnerable service up on `soc-victim`. |
| Automated exploit | The attack, run from `soc-attacker`, reproducible start to finish. |
| Find what happened | The reason the guide exists. The log lines, the Wazuh rule that fired, and what to do when nothing fires. |
| Remediate | The immediate containment step, before a patch is available. |
| Patch | The real fix, and how to verify it took. |

Name the file `NN-short-name-cve-YYYY-NNNNN.md` and add a row to the table in
the README.

### Which CVEs belong here

**Yes:** old, patched, well documented, and reachable with a service you can
stand up yourself in the lab. An exploit that already works reliably leaves the
reader free to concentrate on the detection, which is the part worth learning.

**No:** anything unpatched or recent enough that a working walkthrough is a
weapon. Nothing requiring a licensed or paid product. Nothing that only
reproduces against a live third-party service.

If you are unsure, open an issue and ask before writing it.

## The bar for "find what happened"

A guide that exploits something and stops is not useful here. The detection
section has to show the actual evidence: the raw log line, the rule id that
matched, and the alert as the SIEM renders it. If the stock Wazuh ruleset does
**not** catch it, say so plainly and write the custom rule. A documented gap in
the default ruleset is one of the more valuable things you can contribute.

## Corrections to the build docs

Just as welcome, and lower effort. If a step broke on your Proxmox or Wazuh
version, open an issue with the version and the error. The build docs were
written against one environment and will drift.

## House rules

- Never commit a real credential. The Windows answer files use
  `CHANGEME-LabAdminPassword` and must stay that way.
- Keep lab addressing in `10.99.99.0/24` so guides stay copy-pasteable.
- Test what you write. Every command in a guide should have been run.
- Plain prose, and no em dashes.

## Licence

Contributions are accepted under the MIT licence in [LICENSE](LICENSE).
