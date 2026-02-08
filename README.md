# Fedora Linux – TPM2-Backed Full Disk Encryption (Secure Boot)

This repository documents a **production-ready, reproducible setup** for unlocking a
**LUKS2 full-disk-encrypted Fedora system using TPM 2.0**, while keeping
**UEFI Secure Boot enabled**.

The design goal is simple:

- 🔐 Maintain strong Full Disk Encryption (FDE)
- 🔑 Eliminate manual passphrase entry on normal boots
- 🛡️ Preserve Secure Boot and measured boot guarantees
- 🧯 Retain a passphrase fallback for recovery and maintenance

This setup uses **first-class Fedora tooling** (`systemd-cryptenroll`, `dracut`) — no
third‑party hacks.

---

## System Overview

### Hardware
- **Model:** Dell Precision 5540
- **Firmware:** UEFI
- **TPM:** TPM 2.0
- **Storage:** NVMe SSD

### Operating System
- **Distribution:** Fedora Linux 43
- **Edition:** Workstation
- **Initramfs:** dracut (systemd-based)
- **Boot mode:** UEFI + Secure Boot enabled
- **Disk encryption:** LUKS2 (root filesystem)

---

## Firmware & Platform Prerequisites (Out of Scope)

> **This setup REQUIRES Secure Boot and TPM 2.0 to be enabled in BIOS/UEFI firmware.**

Required firmware state:

- **UEFI Secure Boot:** Enabled and enforcing
- **TPM:** Version 2.0 present and enabled

Configuration steps for enabling Secure Boot, activating TPM, or verifying TPM version
are **device- and vendor-specific** (Dell, Lenovo, HP, firmware revisions, etc.) and are
therefore **out of scope for this document**.

This document assumes that, on the target device:

- Secure Boot is already enabled
- TPM version is **2.0**
- TPM is enabled and visible to the operating system

Verification from Linux:

```bash
ls -l /dev/tpm* /sys/class/tpm/
```

If these prerequisites are not met, **TPM2-based auto-unlock will not function**, and the
system will fall back to manual LUKS passphrase entry.

---

## Security Model

A TPM2-sealed key is enrolled into the LUKS2 container using `systemd-cryptenroll`.

At boot time:

1. Firmware verifies the boot chain (Secure Boot)
2. The TPM measures the boot state into PCRs
3. The TPM releases the disk key **only if measurements match**
4. Fedora unlocks the root filesystem automatically

If measurements do **not** match:

- TPM refuses to release the key
- Fedora prompts for the LUKS passphrase instead

This is expected and intentional behavior.

---

## PCR Policy

- **PCR 7** is used

Why PCR 7:

- Reflects Secure Boot and kernel trust state
- Stable across normal kernel updates
- Detects bootloader or policy tampering
- Recommended balance of security vs usability on Fedora

---

## Implementation Summary

### 1. Enroll TPM2 unlock key
```bash
sudo systemd-cryptenroll   --tpm2-device=auto   --tpm2-pcrs=7   /dev/nvme0n1pX
```

### 2. Enable TPM2 unlocking in crypttab
```text
luks-<LUKS_UUID>  UUID=<LUKS_UUID>  none  discard,tpm2-device=auto
```

### 3. Ensure TPM2 support in initramfs
```bash
echo 'add_dracutmodules+=" tpm2-tss "' | sudo tee /etc/dracut.conf.d/tpm2.conf
sudo dracut -f
```

### 4. Reboot and validate
- Normal boot → auto-unlock
- Unexpected state → passphrase prompt

---

## Recovery & Safety Notes

⚠️ **Always keep at least one traditional LUKS passphrase slot.**

Situations where the passphrase may be required:

- Firmware / BIOS updates
- Secure Boot configuration changes
- TPM reset or replacement
- Disk moved to another system

To remove the TPM2 slot:

```bash
sudo systemd-cryptenroll --wipe-slot=tpm2 /dev/nvme0n1pX
```

---

## Verification Checklist (Sanitised Example Output)

The following example output demonstrates a **correctly configured system**.
All identifiers are anonymised.

### ✅ TPM2 support included in initramfs

```bash
echo 'add_dracutmodules+=" tpm2-tss "' > /etc/dracut.conf.d/tpm2.conf
```

*Ensures TPM libraries are available during early boot.*

---

### ✅ TPM2 auto-unlock enabled in crypttab

```bash
luks-<LUKS_UUID> UUID=<LUKS_UUID> none discard,tpm2-device=auto
```

*Instructs systemd to attempt TPM-based unlock before prompting.*

---

### ✅ LUKS2 confirmed

```bash
Version: 2
```

*`systemd-cryptenroll` requires LUKS2.*

---

### ✅ Disk and filesystem layout

```text
/boot/efi  -> vfat (UEFI)
/boot      -> ext4
/           -> LUKS2 → Btrfs
/home       -> Btrfs subvolume
```

*Typical Fedora Workstation Secure Boot layout.*

---

### ✅ TPM device exposed by kernel

```bash
/dev/tpm0
/dev/tpmrm0
```

*Confirms a functional TPM 2.0 device.*

---

## License

Documentation only.
Free to use, adapt, and improve.
