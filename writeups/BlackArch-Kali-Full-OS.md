---
title: "Kali + BlackArch Dual-Boot (UEFI) — Full Guide"
date: 2025-10-23
summary: "Shared EFI/swap, GRUB nomodeset, iwd/dhcpcd Wi-Fi."
# layout is auto via _config; if you prefer explicit: layout: default
---


---
title: "Kali + BlackArch Dual-Boot (UEFI) — Full Guide"
date: 2025-10-23
summary: "Shared EFI/swap, GRUB nomodeset, iwd/dhcpcd Wi-Fi."
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
7. **GRUB Install:** Skip automatic GRUB installation —we’ll repair manually.  
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

## ⚙️ STEP 8 — Kali + BlackArch Dual-Boot — Full GRUB & EFI Repair Guide (Final Version)

**Goal:** Re-install Kali’s GRUB as the main bootloader, detect BlackArch automatically, and ensure Kali loads first, even on stubborn BIOS systems.

### 1️⃣ Boot into Kali Live
Boot from the Kali Live USB (UEFI mode) and open a terminal.

### 2️⃣ Identify partitions
```bash
lsblk -f
```
Typical layout:
```
/dev/nvme0n1p1 EFI System (vfat)
/dev/nvme0n1p2 Swap (swap)
/dev/nvme0n1p3 BlackArch root (ext4)
/dev/nvme0n1p4 Kali root (ext4)
```

### 3️⃣ Mount Kali & EFI
```bash
sudo umount -R /mnt 2>/dev/null || true
sudo mount /dev/nvme0n1p4 /mnt
sudo mkdir -p /mnt/boot/efi
sudo mount /dev/nvme0n1p1 /mnt/boot/efi
sudo mount --bind /dev /mnt/dev
sudo mount --bind /proc /mnt/proc
sudo mount --bind /sys /mnt/sys
cat /mnt/etc/os-release
```
(Should show NAME="Kali GNU/Linux")

### 4️⃣ Chroot into Kali
```bash
sudo chroot /mnt /bin/bash
```

### 5️⃣ Install GRUB and tools
```bash
apt update
apt install -y grub-efi-amd64 efibootmgr os-prober
```

### 6️⃣ Enable OS detection
```bash
grep -q '^GRUB_DISABLE_OS_PROBER=false' /etc/default/grub || echo 'GRUB_DISABLE_OS_PROBER=false' >> /etc/default/grub
```

### 7️⃣ Reinstall and rebuild GRUB
```bash
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=Kali --recheck
update-grub
```
(You should see “Found Arch Linux on /dev/nvme0n1p3”)

### 8️⃣ Mount pseudo-terminals & EFI variables
```bash
mount -t devpts devpts /dev/pts
mount -t efivarfs efivarfs /sys/firmware/efi/efivars
```

### 9️⃣ Delete the rogue Linux Boot Manager (BlackArch’s entry)
```bash
efibootmgr -v
efibootmgr -b 0003 -B (replace 0003 with the ID shown for "Linux Boot Manager")
rm -rf /boot/efi/EFI/systemd
```

### 🔟 Re-register Kali’s GRUB and set boot order
```bash
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=Kali --recheck
efibootmgr -o 0000,0006,0007 (adjust the numbers so Kali is first)
efibootmgr -v
```

### 11️⃣ Hard-wire the BIOS fallback (forces BIOS to load Kali GRUB)
```bash
mkdir -p /boot/efi/EFI/BOOT
cp /boot/efi/EFI/kali/grubx64.efi /boot/efi/EFI/BOOT/bootx64.efi
```

Check:
```bash
ls /boot/efi/EFI/BOOT/
```
(Should show bootx64.efi)

### 12️⃣ Optional cleanup of leftover EFI folders
```bash
rm -rf /boot/efi/EFI/arch
rm -rf /boot/efi/EFI/linux
```

### 13️⃣ Final verification & reboot
```bash
efibootmgr -v
exit
sudo reboot
```
Remove the USB stick before the reboot so BIOS loads from disk.

**Expected result:**
Kali’s GRUB menu appears first every time.
GRUB lists both Kali GNU/Linux and Arch Linux (BlackArch).
BlackArch boots only when selected, and Kali remains default.

---

## 🧠 STEP 9 — How to Create a Permanent “nomodeset” GRUB Entry for BlackArch Linux (from Kali)

**GOAL:**
If BlackArch boots successfully from Kali’s GRUB menu but only when you manually type `nomodeset`, this step will show you how to create a permanent GRUB entry so you never have to enter it again.

1️⃣ Confirm GRUB Detects BlackArch
```bash
sudo update-grub
```
You should see output similar to:
```
Found Arch Linux on /dev/nvme0n1p3
```
If you can boot BlackArch from this menu entry, continue.

2️⃣ Find the Kernel Lines Used by GRUB
```bash
grep -A5 "menuentry 'Arch" /boot/grub/grub.cfg
```
You’ll see output like:
```
menuentry 'Arch Linux (rolling) (on /dev/nvme0n1p3)' {
   insmod part_gpt
   insmod fat
   search --no-floppy --fs-uuid --set=root EB9B-4F74
   linux /vmlinuz-linux root=/dev/nvme0n1p3
   initrd /initramfs-linux.img
}
```

3️⃣ Create a Custom GRUB Entry
```bash
sudo nano /etc/grub.d/40_custom
```
Paste and edit as needed:
```
menuentry 'BlackArch (nomodeset) (on /dev/nvme0n1p3)' --class gnu-linux --class gnu --class os {
   insmod part_gpt
   insmod fat
   search --no-floppy --fs-uuid --set=root EB9B-4F74
   linux /vmlinuz-linux root=/dev/nvme0n1p3 rw nomodeset
   initrd /initramfs-linux.img
}
```
Save (**Ctrl+O**, **Enter**, **Ctrl+X**).

4️⃣ Rebuild GRUB
```bash
sudo update-grub
```

5️⃣ Reboot and test
You’ll now see a new entry in the GRUB menu:
```
BlackArch (nomodeset) (on /dev/nvme0n1p3)
```

6️⃣ Make the New Entry the Default (Optional)
```bash
grep -n "menuentry '" /boot/grub/grub.cfg
sudo grub-set-default "BlackArch (nomodeset) (on /dev/nvme0n1p3)"
sudo update-grub
```

7️⃣ Summary
You copied your existing working BlackArch GRUB entry, added `nomodeset`, and made it permanent. This persists across updates.

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
