---
title: What is initrd?
date: 2026-04-02
categories:
  - Jetson Orin Nano
  - Deep Dive
---

# What is initrd?

If you have ever peeked at `/boot/extlinux/extlinux.conf`, you probably noticed a line called `INITRD`. Have you ever wondered what this thing is, what it does, and why we need it?

<!-- more -->

---

!!! note "Note"
    These notes are based on NVIDIA Jetson Linux R36.5 behavior and the `nvidia-l4t-initrd` package.

## Why do we need initrd?

After UEFI finishes, the kernel is launched with three key inputs:

- **`Image`**: The kernel binary.
- **`FDT`**: The device tree blob (DTB), which describes the hardware.
- **`INITRD`**: <--- This is what we're going to talk about today!

You can see them in `extlinux.conf`:

```sh
LABEL primary
    LINUX /boot/Image
    FDT /boot/dtb/kernel_tegra234-p3768-0000+p3767-0005-nv.dtb
    INITRD /boot/initrd
    APPEND ${cbootargs} root=PARTUUID=<...> rw rootwait rootfstype=ext4
```

Even with `Image` and `FDT`, the kernel needs to know "how" to mount the root partition (e.g., NVMe, PCIe, ext4). 

Who knows the "how"? The **drivers**! 
But where are the drivers? They are usually compiled as modules (`.ko` files), which live on the root partition itself!

See this chicken-and-egg problem? This is why **initrd** exists. 

The initrd breaks the cycle: the bootloader loads it into RAM, so the kernel can access the modules *before* it mounts the disk.

!!! note
    In one sentence: The **initrd** (initial ramdisk) is a small, compressed filesystem loaded into RAM by the bootloader. It provides the kernel with the drivers it needs to access the real root filesystem.

---

## What is inside initrd?

On Jetson, `/boot/initrd` is a gzipped `cpio` archive. The files included in the initrd are specified via configuration lists in:

```bash
/etc/nv-update-initrd/list.d/
```

The `modules` list includes both in-tree and out-of-tree (OOT) modules:

```text
# nvidia-l4t-initrd deb: /etc/nv-update-initrd/list.d/modules

# --- Out-of-Tree (OOT) Modules (from /lib/modules/<VER>/updates/) ---
nvethernet.ko        # NVIDIA ethernet driver
nvpps.ko             # Pulse-per-second driver
r8126.ko             # Realtek 8126 ethernet
r8168.ko             # Realtek 8168 ethernet
fusb301.ko           # USB Type-C controller

# --- In-Tree Modules (from /lib/modules/<VER>/kernel/) ---
nvme.ko              # NVMe host driver
nvme-core.ko         # NVMe core
tegra-bpmp-thermal.ko # Thermal management
pwm-tegra.ko         # PWM controller (for fan)
pwm-fan.ko           # Fan control
pcie-tegra194.ko     # PCIe controller
phy-tegra194-p2u.ko  # PCIe PHY
tegra-xudc.ko        # USB device controller
typec_ucsi.ko        # USB Type-C UCSI
ucsi_ccg.ko          # USB Type-C CCG
typec.ko             # USB Type-C subsystem
uas.ko               # USB Attached SCSI
```

The `binlist` includes binaries for disk encryption (LUKS) and secure storage:

```text
# nvidia-l4t-initrd deb: /etc/nv-update-initrd/list.d/binlist

/usr/sbin/nvluks-srv-app    # Disk Encryption
/usr/sbin/tee-supplicant    # Secure Storage
/usr/lib/libteec.so.2       # TEE client library
```

!!! important
    Now that we know what's inside, it's clear that the **initrd must stay in sync** with our modules. If we ever modify or update modules (either in-tree or OOT), we better update the initrd as well!

### How to update initrd?

We use the `nv-update-initrd` command:

```bash
$ sudo nv-update-initrd
```

Let's dive into what `nv-update-initrd` actually **does**.

The script lives at `/usr/sbin/nv-update-initrd` on the Jetson, installed by the `nvidia-l4t-initrd` Debian package. Its job is to repack the `/boot/initrd` file with fresh modules that match our current kernel.

Here is exactly what happens when we run `sudo nv-update-initrd`, step by step:

### Step 1: Backup the current initrd
```bash
cp "${INITRD}" "${INITRD_BAK}"
```
Before making any changes, the script clones the current working initrd file. If the update fails mid-way, we have a "last known good" version to restore.

### Step 2: Unpack the initrd
```bash
DESTDIR=$(mktemp -d)
pushd "${DESTDIR}"
gunzip -c "${_initrd_path}" | cpio -i --quiet
```
The initrd is essentially a compressed archive (a `.cpio.gz` file). The script creates a temporary folder and unpacks everything there so it can modify the files. After this step, `$DESTDIR` looks like a mini Linux filesystem with `lib/`, `etc/`, `usr/`, etc.

### Step 3: Remove old modules
```bash
rm -rf "${DESTDIR}/lib/modules/"*
```
Kernel modules take up significant space. To prevent the initrd from growing uncontrollably with every update, the script wipes out all old drivers before adding fresh ones.

### Step 4: Identify the kernel version
```bash
_kernel_version="$(strings "${_image_path}" \
    | grep -oE "Linux version [0-9a-zA-Z\.\-]+[+]* " \
    | cut -d\  -f 3 | head -1)"
```
The script looks at the actual kernel binary (`/boot/Image`) and extracts the version string (e.g., `5.15.185-tegra`). It needs this exact string to know which folder under `/lib/modules/` to pull the new drivers from.

### Step 5: Inject new drivers
```bash
# Read the list of "mission-critical" modules
for file in "${LIST_DIR}/"*; do
    cat "${file}" >> "${temp_list}"
done

# Copy each listed module into the initrd workspace
copy_files_initrd "${DESTDIR}" "${temp_list}" "${_kernel_version}"
```
It reads the configuration files in `/etc/nv-update-initrd/list.d/` (the `modules` and `binlist` files shown previously) to decide which drivers are mission-critical for booting. Then it pulls those fresh `.ko` files from our system's `/lib/modules/<version>/` into the temporary folder.

### Step 6: Copy modprobe configs
```bash
for modprobe_dir in "/etc/modprobe.d" "/lib/modprobe.d"; do
    rm -rf "${DESTDIR:?}/${modprobe_dir}"
    mkdir -p "${DESTDIR}/${modprobe_dir}"
    for file in "${modprobe_dir}"/*.conf; do
        copy_file config "$file"
    done
done
```
It copies our custom hardware rules (`modprobe` configs). These files tell the system how to load certain drivers (e.g., specific parameters for the GPU or Wi-Fi).

### Step 7: Repacking the archive
```bash
find . | cpio -H newc -o --quiet | gzip -9 -n > "${_initrd_path}"
```
Now that the files are updated, the script zips the folder back up into the `cpio.gz` format.

### Step 8: The final swap
```bash
# Recall from Step 1: INITRD_BAK is the file we've been modifying
cp "${INITRD_BAK}" "${INITRD}"
rm -f "${INITRD_BAK}"
```

---

## Conclusion

1.  **Booting**: At the beginning of the boot process, the bootloader (UEFI) reads `extlinux.conf`, sees the `INITRD /boot/initrd` line, and copies the entire initrd file into RAM. Now the kernel has a temporary filesystem in memory containing all those critical `.ko` files and modprobe configs before the SSD is even accessible.

2.  **The kernel uses the initrd**: It unpacks the ramdisk, loads the NVMe/PCIe drivers from it, and uses those drivers to mount the real root filesystem on the SSD.

3.  **The rest loads from disk**: Once the SSD is mounted, `systemd` takes over and loads all the remaining modules (WiFi, Bluetooth, camera, sound, etc.) from `/lib/modules/<version>/` on the real filesystem.

That's it! That is the initrd. I got so excited after understanding how initrd works, and I hope you do as well!
