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

## Multilingual Keyboard Layout Considerations (Early Boot)
On systems configured with **multiple keyboard layouts**, early boot environments
(initramfs, LUKS prompt) may not reflect the desktop keymap.

### Mitigation strategies

- Use **ASCII-only LUKS passphrases**
- Configure console keymaps explicitly if required
- Keep at least one fallback unlock method
- If a new kernel fails to unlock, **boot a previous known-working kernel**,
  which uses its own initramfs and keymap

On systems configured with **multiple keyboard layouts** (e.g. US + non-US layouts),
it is important to understand that **early boot environments** (initramfs, LUKS prompt)
do **not always reflect the desktop keyboard configuration**.

In particular:

- The **LUKS passphrase prompt** during early boot typically uses the **default keymap**
  embedded in the initramfs
- Graphical desktop layout switching (e.g. `Win + Space`) is **not available**
- After installation or **after early kernel updates**, the active keymap may
  temporarily revert to the default (commonly `us`)

This can result in:
- Passphrases appearing “incorrect” when typed
- Unlock failures immediately after installation
- Unlock failures after the first kernel or initramfs rebuild

This behavior is expected and documented.

### Mitigation strategies

- Prefer a **layout-agnostic passphrase** (ASCII-only) for LUKS
- Explicitly configure the console keymap if relying on non-US layouts
- Always retain at least one fallback unlock method (TPM2 or passphrase)
- If a new kernel fails to unlock, **boot a previous known-working kernel**,
  which uses its own initramfs and keymap

For details and discussion, see Fedora’s official community guidance:
[5]

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

## Example and Verification Checklist: 

The following example output demonstrates a **correctly configured system**.
All identifiers are anonymised.

### ✅ TPM2 support included in initramfs

⚠️  Check first! on Fedora Linux 43 (Workstation Edition) that is default:

```bash
cat /etc/dracut.conf.d/tpm2.conf
# expected output:
add_dracutmodules+=" tpm2-tss "
```

If not, you can add it:

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

## Sources & References

[1] Fedora Magazine – TPM2 auto-unlock with systemd  
  https://fedoramagazine.org/use-systemd-cryptenroll-with-fido-u2f-or-tpm2-to-decrypt-your-disk/

[2] Fedora Magazine – Automatically decrypt your disk using TPM2  
  https://fedoramagazine.org/automatically-decrypt-your-disk-using-tpm2/

[3] systemd documentation – systemd-cryptenroll(1)  
  https://www.freedesktop.org/software/systemd/man/systemd-cryptenroll.html

[4] Fedora Documentation – UEFI Secure Boot  
  https://jfearn.fedorapeople.org/fdocs/en-US/Fedora_Draft_Documentation/0.1/html-single/UEFI_Secure_Boot_Guide/index.html

[5] Fedora Discussion – Real-world TPM2 + LUKS setups  
  https://discussion.fedoraproject.org/

[6] Fedora Discussion – Keyboard layout issues at LUKS prompt  
https://discussion.fedoraproject.org/t/how-to-change-layout-in-luks-passphrase/145687

---

## License

Documentation only.
Free to use, adapt, and improve.
