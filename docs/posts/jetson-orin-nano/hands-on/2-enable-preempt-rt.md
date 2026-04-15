---
title: Enable PREEMPT_RT
date: 2026-03-22
categories:
  - Jetson Orin Nano
  - Hands-on
---

# Enable PREEMPT_RT

This guide covers how to enable **PREEMPT_RT** on the Jetson Orin Nano, transforming it into a real-time operating system (RTOS) ready for low-latency robotics applications.

<!-- more -->

---

!!! note "Note"
    These notes are based on the official [NVIDIA Jetson Linux Developer Guide R36.5 - Kernel Customization](https://docs.nvidia.com/jetson/archives/r36.5/DeveloperGuide/SD/Kernel/KernelCustomization.html).

Watch the following video for a complete walkthrough of the process:
[YouTube Video Link] (xxx)

## 1. Verify system performance

Before we jump into the build, let's establish a performance baseline. This helps us see the actual real-time improvement later on.

### Check the current kernel

First, let's verify the kernel version and variant currently running on the device.

```bash hl_lines="2"
$ uname -a
# Linux ubuntu 5.15.136-tegra #1 SMP PREEMPT Tue Jan 13 01:23:45 UTC 2026 aarch64 aarch64 aarch64 GNU/Linux
```

### Install stress-test tools

Install the necessary utilities for real-time performance measurement and system stressing.

```bash
$ sudo apt update
$ sudo apt upgrade -y

# Install RT and stress testing tools
$ sudo apt install -y rt-tests stress-ng
```

### Run latency tests

To measure the baseline latency, run a stress test in one terminal and the cyclic test in another.

1.  **Terminal 1**: Start the CPU and I/O stress:
    ```bash
    $ sudo stress-ng --cpu 8 --io 4 --vm 2 --vm-bytes 1G --timeout 30s
    ```

2.  **Terminal 2**: Run the latency measurement:
    ```bash
    $ sudo cyclictest --mlockall --smp --priority=80 --interval=200 --duration=10m
    ```

!!! example "Example Output"
    ![Latency Test Results](../img/2-enable-preempt-rt-1-stress-test.png)

---

## 2. Prerequisites

Make sure to have the source code and packages ready before moving forward. Check out [Get Ready!](./1-get-ready.md)

---

## 3. Build and enable PREEMPT_RT

Now for the fun part --> building and enabling the patch! We'll use a helper repository to streamline the process.

Download the helper repository and run the script.

### Clone the helper repository

```bash
$ git clone git@github.com:rylanpeng/jetson-orin-nano-preempt-rt.git
```

### Run the build script

```bash
$ cd jetson-orin-nano-preempt-rt
$ ./build-preempt-rt.sh <install-path>
```

!!! note
    The **`<install-path>`** should point to the root directory where the source code was extracted. The script will look for files within `<install-path>/Linux_for_Tegra/source/`.

---

## 4. Verify the installation

Once the script finishes, it's a good idea to double-check that everything was installed correctly.

### Inspect the /boot directory

We should see **`Image.backup`** and **`initrd.backup`** files created alongside the new kernel image.

```bash
$ ls /boot
# Image  Image.backup  initrd  initrd.backup  ...
```

### Inspect kernel modules

Check `/lib/modules` to ensure both the **standard** and **RT** versions of the kernel modules are present.

```bash
$ ls /lib/modules
# 5.15.136-tegra  5.15.136-rt-tegra
```

---

## 5. Add a fallback boot entry

### Edit extlinux.conf

Open the boot configuration file with `vim` or preferred text editor.

```bash
$ sudo vim /boot/extlinux/extlinux.conf
```

### Add the backup entry

Modify the configuration to include a fallback option:

1.  **Uncomment** the `backup` section.
2.  **Copy** the `APPEND` line from the current default section to the `backup` section.
3.  **Update** the `INITRD` path to point to `/boot/initrd.backup`.

!!! tip "Example Configuration"
    ```sh hl_lines="3-4"
    LABEL backup
        MENU LABEL backup kernel
        LINUX /boot/Image.backup
        INITRD /boot/initrd.backup
        APPEND ${cbootargs} root=PARTUUID=cb24a353-d890-4252-9751-21d939088c7c rw rootwait rootfstype=ext4 mminit_loglevel=4 console=ttyTCU0,115200 firmware_class.path=/etc/firmware fbcon=map:0 video=efifb:off console=tty0 efi=runtime pci=pcie_bus_perf nvme.use_threaded_interrupts=1 nv-auto-config
    ```

---

## 6. Reboot
```sh
$ sudo reboot
```

---

## 7. Verify the boot selection

During the boot process, we will be presented with the boot menu.

Select the **backup** entry if you ever need to roll back to the original kernel; otherwise, the system will boot using the new **PREEMPT_RT** image by default.

---

## 8. Verify the results

### Check the RT kernel variant

Run `uname -a` again, and you should see the **`PREEMPT_RT`** identifier!

```bash hl_lines="2"
$ uname -a
# Linux ubuntu 5.15.136-rt-tegra #1 SMP PREEMPT_RT Tue Jan 13 01:23:45 UTC 2026 aarch64 aarch64 aarch64 GNU/Linux
```

### Re-run latency tests

Perform the stress test and `cyclictest` again to observe the improvement in latency.

!!! example "Example Output"
    ![Latency Test Results (RT)](../img/2-enable-preempt-rt-2-stress-test.png)

---

## Optimizing performance (Optional)

You can boost performance even further by increasing the kernel timer frequency (`CONFIG_HZ`).

### 1. Open defconfig

```sh
$ vim Linux_for_Tegra/source/kernel/kernel-jammy-src/arch/arm64/configs/defconfig
```

### 2. Update configuration
```bash
CONFIG_HZ_1000=y
```

### 3. Build and install again

#### Restore original images

Restore the original `Image` and `initrd` before rebuilding. This ensures the script's backup step saves the non-RT kernel as the fallback, not the RT kernel currently in `/boot`.

```bash
$ sudo cp /boot/Image.backup /boot/Image
$ sudo cp /boot/initrd.backup /boot/initrd
```

#### Re-run the build script

```bash
$ ./build-preempt-rt.sh <install-path>
```

### 5. Reboot

Finally, reboot the system to apply the changes.

```bash
sudo reboot
```
