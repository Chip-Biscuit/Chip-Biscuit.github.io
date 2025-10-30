---
title: "Kali + BlackArch Dual-Boot (UEFI) — Full Guide"
date: 2025-10-23
summary: "Shared EFI/swap, GRUB nomodeset, iwd/dhcpcd Wi-Fi."
# layout is auto via _config; if you prefer explicit: layout: default
---


# ⚙️ Kali + BlackArch Dual-Boot — Full Bare-Metal Installation & EFI Repair Guide

## 🖥️ System Overview
**Laptop Model:** MSI Cyborg 15 A12UDX (REV 1.0)  
**CPU:** Intel Core i5-12450H (12 cores @ 4.40 GHz)  
**GPU (Discrete):** NVIDIA GeForce RTX 3050 6 GB Laptop GPU  
**GPU (Integrated):** Intel UHD Graphics @ 1.20 GHz  
**Memory:** 16 GB DDR5  
**Storage:** 500 GB NVMe SSD (target drive for dual-boot installation)  
**Display:** 15.6″ 1920×1080 @ 144 Hz (CMN1521)  
**Firmware Mode:** UEFI with Secure Boot disabled  

---

## 🪪 STEP 1 — Preparing Installation Media (Ventoy)
**Goal:** Create a multi-boot USB containing Kali Full, Kali Live and BlackArch Full ISOs.

1. Boot into a Kali Live session.  
2. Install Ventoy:  
   ```bash
   sudo apt install ventoy
   ```  
3. Identify the USB device:  
   ```bash
   lsblk
   ```  
4. Install Ventoy onto the USB (16 GB minimum):  
   ```bash
   sudo ventoy -i /dev/sdX
   ```  
   *(replace `sdX` with your USB device)*  
5. Copy these ISOs to the Ventoy drive:  
   - **Kali Linux Full Installer**  
   - **Kali Linux Live** (for later GRUB repairs)  
   - **BlackArch Full Installer**  
6. In BIOS, set USB boot priority above the NVMe drive.  

**Result:** A single Ventoy stick capable of booting both installers and a live rescue environment.  

---

## ⚙️ STEP 2 — Booting into BlackArch Installer (Hybrid Graphics)
**Goal:** Bypass hybrid-GPU display issues when booting BlackArch.

1. Boot from Ventoy → select **BlackArch Full ISO**.  
2. Highlight the default install option but press `E`.  
3. Move to the end of the kernel line and append:  
   ```
   nomodeset
   ```  
4. Press **Ctrl + X** or **F10** to boot.  

**Result:** BlackArch boots without display errors.  
*(You can remove `nomodeset` later after installing drivers.)*  

---

## 🔑 STEP 3 — Logging into the BlackArch Live Environment
At the login prompt:  
```
Username: root
Password: blackarch
```
You’re now in the root environment and can open a terminal to begin installation.  

---

## 💽 STEP 4 — Launching the BlackArch Installer
Open a terminal and start the installer:  
```bash
blackarch-install
```
Follow the text interface to begin setup.  

---

## 🧩 STEP 5 — Installing BlackArch Linux (Manual Partitioning)
**Goal:** Install BlackArch and prepare space for Kali.  

1. **Installer Mode:** Choose `2 → Install from ISO`, then `2 Verbose`.  
2. **Locale & Keymap:** `en_GB.UTF-8`, keymap `uk`.  
3. **Hostname:** `chip`.  
4. **Partitioning:**  
   - Choose manual with `cfdisk`.  
   - Table type: **GPT**.  
   - Layout:  

| Mount | Size | Type | Notes |
|:--|:--|:--|:--|
| /boot/efi | 1 GB | EFI System | shared with Kali |
| [SWAP] | 4 GB | Linux swap | shared swap |
| / (root) | 250 GB | Linux filesystem | BlackArch root |
| Free | 221.9 GB | Linux filesystem | reserved for Kali |

   - Write changes and quit.  
5. **Filesystem Setup:**  
   - EFI → FAT32 mount `/boot/efi`  
   - Root → ext4 mount `/`  
   - Swap → as swap  
6. **No encryption.**  
7. **Set root password**, optional user, choose timezone `Europe/London`.  
8. **Clock:** Use UTC.  
9. Wait for install to complete, then reboot.  

**Result:** BlackArch installed with shared EFI and swap.  

---

## 🐉 STEP 6 — Installing Kali Linux Alongside BlackArch
**Goal:** Install Kali on the remaining space and share EFI/swap.  

1. Boot from Ventoy → **Kali Everything ISO** → **Graphical Install**.  
2. Language: English → United Kingdom → British English.  
3. Configure network and create user.  
4. Choose **Manual Partitioning**.  

| Partition | Size | Action |
|:--|:--|:--|
| #1 (1 GB) | Mount `/boot/efi`, bootable ON, no format |
| #2 (4 GB) | Use as swap area, no format |
| #3 (250 GB) | BlackArch root — leave untouched |
| #4 (221.9 GB) | Use as ext4, mount `/`, format yes |

5. Finish partitioning → write changes.  
6. **Software Selection:** Choose desktop environment (XFCE, GNOME or KDE Plasma). *(KDE Plasma used here.)*  
   Select almost all tools for a complete setup.  
7. **GRUB Install:** Skip automatic GRUB installation — we’ll repair manually.  
8. **Clock:** Set to UTC.  
9. Complete installation and reboot.  

**Result:** Kali installed successfully on its own partition.  
The system will still boot directly into BlackArch for now.  

---

## ⚙️ STEP 7 — Preparing to Repair and Configure GRUB
1. Reboot to confirm it boots into BlackArch (default behavior).  
2. Power off and insert the Ventoy USB again.  
3. Boot into **Kali Live (UEFI mode)** → open a terminal.  

**Result:** You’re in Kali Live and ready to repair GRUB.  

---

## ⚙️ STEP 8 — Kali + BlackArch Dual-Boot — Full GRUB & EFI Repair Guide
**Goal:** Re-install Kali’s GRUB as the main bootloader, detect BlackArch automatically, and make Kali load first.  

Follow the complete GRUB and EFI repair process as detailed above.  

---

## 🧠 STEP 9 — Create a Permanent “nomodeset” GRUB Entry for BlackArch (from Kali)
**Goal:** Keep the `nomodeset` parameter permanently for hybrid-GPU systems.  

1️⃣ **Verify BlackArch detected:**  
```bash
sudo update-grub
```
Confirm you see “Found Arch Linux on /dev/nvme0n1p3”.  

2️⃣ **Find the existing entry:**  
```bash
grep -A5 "menuentry 'Arch" /boot/grub/grub.cfg
```
Note the UUID and kernel/initrd paths.  

3️⃣ **Create custom entry:**  
```bash
sudo nano /etc/grub.d/40_custom
```
Add to the end:  
```
menuentry 'BlackArch (nomodeset) (on /dev/nvme0n1p3)' --class gnu-linux --class gnu --class os {
   insmod part_gpt
   insmod fat
   search --no-floppy --fs-uuid --set=root EB9B-4F74
   linux /vmlinuz-linux root=/dev/nvme0n1p3 rw nomodeset
   initrd /initramfs-linux.img
}
```
Save and exit: **Ctrl + O**, **Enter**, **Ctrl + X**.  

4️⃣ **Rebuild GRUB:**  
```bash
sudo update-grub
```

5️⃣ **Reboot and test:** Select “BlackArch (nomodeset)” in GRUB.  

6️⃣ **Make default if desired:**  
```bash
grep -n "menuentry '" /boot/grub/grub.cfg
sudo grub-set-default "BlackArch (nomodeset) (on /dev/nvme0n1p3)"
sudo update-grub
```

**Result:** BlackArch boots permanently with `nomodeset` applied, no manual editing needed.  

---

## ✅ Final Outcome
- Kali and BlackArch both installed on a single 500 GB NVMe SSD.  
- Shared EFI and swap partitions.  
- Consistent hardware clock (UTC).  
- Stable UEFI boot with Kali’s GRUB managing both systems.  
- Permanent `nomodeset` entry for BlackArch on hybrid graphics laptops.  

---

### 🧭 End of Guide
You’ve now completed a fully manual, UEFI-compliant dual-boot installation with custom bootloader control and hybrid-GPU support.  
This is a rock-solid template for any future multi-OS penetration-testing lab build.  
