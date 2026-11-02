# Building the Windows VMs

Both Windows endpoints (Server 2025 and Windows 11 Enterprise) were installed
fully headless from Microsoft's official evaluation ISOs, driven remotely with
no console access beyond screenshots and keystroke injection. Getting there
surfaced three distinct, non-obvious bugs. They are documented here because the
debugging is the interesting part.

## The approach

- OVMF (UEFI) firmware, required for modern Windows.
- `autounattend.xml` answer files for unattended setup (the Windows equivalent
  of `cloud-init`), attached as a second CD-ROM. See the `.example` files in
  `config/windows/`.
- A vTPM 2.0 device and Secure Boot with pre-enrolled keys for the Windows 11
  box, to satisfy its hardware requirements.

## Bug 1: the rebuilt "no-prompt" ISO was missing UDF

Microsoft install media shows an interactive "Press any key to boot from CD or
DVD" prompt. In a headless build there is no reliable way to land that single
keystroke in its short window, so the ISO was rebuilt to use Microsoft's own
`efisys_noprompt.bin` boot image, which boots straight through.

The first rebuild used `genisoimage`/`xorriso` with ISO9660 + Joliet but
**dropped the UDF filesystem**. That looked fine, until every boot failed the
same way: WinPE started to load, then silently reset, over and over, never
touching the disk.

Root cause: `install.wim` on these ISOs is **7.3 GB**, well past the ~4 GB
per-file limit of ISO9660. UDF exists specifically to carry files that large.
Without it, the bootloader could not read the image and faulted mid-load. The
giveaway was comparing the two ISOs:

```
original ISO:        filesystem = udf,      install.wim = 7.3 GB  (readable)
rebuilt ISO:         filesystem = iso9660,  install.wim = 7.3 GB  (unreadable by Windows)
```

Fix: rebuild with `genisoimage -udf -allow-limited-size`, keeping the
`efisys_noprompt.bin` boot image. The filesystem then matched the original and
WinPE loaded cleanly.

## Bug 2: wrong VirtIO driver for the controller

With good media, Setup reached "Select location to install Windows" but showed
an **empty disk list**: "Hardware not showing up? Load driver to access your
hardware." The disk controller was configured as `virtio-scsi`, so the VirtIO
storage driver has to be loaded during setup (Windows has no built-in driver
for it).

The first attempt loaded `viostor` and it reported success, but the disk still
did not appear. `viostor` is the driver for **virtio-blk**, a different device.
The driver for a **virtio-scsi** controller is `vioscsi`. Loading the correct
one (`drvload e:\vioscsi\<os>\amd64\vioscsi.inf`) made the disk visible at the
OS level immediately, confirmed with `wmic diskdrive list brief` showing
`\\.\PHYSICALDRIVE0`.

The VirtIO driver ISO (`virtio-win.iso`, from the official Fedora/Red Hat
project) is attached as a third CD-ROM so the driver is available during setup.

## Bug 3: Setup's GUI would not refresh its disk list

Even with the correct driver loaded and the disk visible from the command line
(`wmic diskdrive` saw an 80 GB `PHYSICALDRIVE0` with 0 partitions), Setup's
graphical disk picker never repopulated. Its Refresh control did not pick up
the newly-available disk.

Rather than keep fighting the GUI, the install was completed by hand from a
Setup command prompt (`Shift+F10`), running exactly the sequence Setup performs
internally:

```bat
diskpart
  select disk 0
  clean
  convert gpt
  create partition efi size=100
  format quick fs=fat32
  assign letter=s
  create partition msr size=16
  create partition primary
  format quick fs=ntfs
  assign letter=c
  exit

:: apply the OS image
dism /apply-image /imagefile:d:\sources\install.wim /index:2 /applydir:c:\

:: inject the storage driver into the OFFLINE image (critical, or it won't boot)
dism /image:c:\ /add-driver /driver:e:\vioscsi\<os>\amd64

:: place the answer file for first-boot config
copy f:\autounattend.xml c:\unattend.xml

:: write the UEFI bootloader
bcdboot c:\windows /s s: /f uefi
```

On reboot, the machine came up off the real disk, ran through the `specialize`
and `oobeSystem` passes, applied the hostname and `FirstLogonCommands` from the
answer file (enabling WinRM, RDP, and ICMP), pulled a DHCP lease under its
intended hostname, and was reachable on 5985 (WinRM) and 3389 (RDP).

The same recipe, with the `w11` driver folder and `index:1`, built the
Windows 11 box.

## A note on driving this headlessly

With no interactive console, keystrokes were injected via QEMU's monitor
(`qm sendkey` / `sendkey` over `qm monitor`) and progress was read from
periodic screenshots (`screendump`). Two things that mattered:

- Under host load, each `qm sendkey` process took ~2 s to spawn. Batching
  multiple keys into one `qm monitor` session, with short pacing between small
  groups, was far faster and far more reliable than one process per key.
- Verification was always against ground truth (`wmic`, partition tables, DHCP
  leases, open ports), never against how a screen "looked," because Setup's GUI
  state proved untrustworthy.
