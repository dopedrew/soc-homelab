# Windows answer files

`autounattend.xml` files for fully unattended Windows setup, the Windows
equivalent of cloud-init. Attach one as a second CD-ROM alongside the Windows
installer ISO.

**Passwords are redacted.** Every `<Value>` password field reads
`CHANGEME-LabAdminPassword`. Set your own before use. These are lab-only
credentials for machines on an isolated network with no path to anything real;
even so, do not commit a real password back into the repo.

The driver-injection paths assume the VirtIO driver ISO
(`virtio-win.iso`, from the official Fedora/Red Hat virtio-win project) is
attached as an additional CD-ROM. See `docs/build-windows-vms.md`.
