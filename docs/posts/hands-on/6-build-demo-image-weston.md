---
title: Build demo-image-weston for Jetson Orin Nano
date: 2026-05-18
categories:
  - Jetson Orin Nano
  - Yocto
  - Hands-on
---

# Build `demo-image-weston` for Jetson Orin Nano

`demo-image-weston` is the first image in the `tegra-demo-distro` ladder where DisplayPort works out of the box.

This post walks through the host setup, Yocto build, flash flow, and first boot checks for a Jetson Orin Nano booting from NVMe.

<!-- more -->

## 1. Install Host Packages

Start with the normal Yocto host dependencies. These packages cover the compiler toolchain, Python modules, archive tools, device tree compiler, and helper tools used during the build.

```bash
$ sudo apt install gawk wget git diffstat unzip texinfo gcc build-essential \
  chrpath socat cpio python3 python3-pip python3-pexpect xz-utils \
  debianutils iputils-ping python3-git python3-jinja2 libegl-mesa0 \
  libsdl1.2-dev pylint xterm python3-subunit mesa-common-dev zstd liblz4-tool \
  device-tree-compiler
```

## 2. Disable GNOME Automount

If your host is running Ubuntu 24.04, disable GNOME automount before flashing. During `initrd-flash`, the Jetson NVMe appears on the host as a USB mass storage device. The flash script partitions that device, asks the kernel to reread the partition table, and writes each partition image.

GNOME automount can grab the new partitions as soon as they appear. When that happens, `partprobe` can fail because the partition is already mounted.

```bash
$ gsettings set org.gnome.desktop.media-handling automount false
$ gsettings set org.gnome.desktop.media-handling automount-open false
```

## 3. Clone tegra-demo-distro

```bash
$ git clone https://github.com/OE4T/tegra-demo-distro.git
$ cd tegra-demo-distro
$ git submodule update --init --recursive
```

The submodule command pulls BitBake, OpenEmbedded Core, meta-tegra, and the other layers required by the distro.

## 4. Initialize the Weston Build

This blog post is assuming we're using NVMe, so initialize the build with the NVMe machine and the `tegrademo` distro.

```bash
$ cd ~/tegra-demo-distro
$ source setup-env --machine jetson-orin-nano-devkit-nvme --distro tegrademo build-weston
```

`setup-env` writes `MACHINE` and `DISTRO` into `build-weston/conf/local.conf`. You do not need to set them again by hand.

## 5. Edit local.conf

Open `build-weston/conf/local.conf` and add these lines at the bottom.

```sh
# Enable Wayland and OpenGL across recipes.
DISTRO_FEATURES:append = " wayland opengl"

# Accept the NVIDIA commercial BSP license.
LICENSE_FLAGS_ACCEPTED = "commercial_tegra-binaries"

# Delete recipe work directories after packaging to save disk space.
INHERIT += "rm_work"
```

The `rm_work` setting is optional, but I recommend it for this build. The Weston image can use a large amount of disk space. `rm_work` deletes each recipe work directory after packaging, while keeping downloads and shared state cache for future builds.

## 6. Build demo-image-weston

Run BitBake from the initialized shell.

```bash
$ bitbake demo-image-weston
```

The first build can take several hours. Weston, Wayland, the NVIDIA GL stack, GStreamer, and the display test packages add much more work than a minimal console image.

## 7. Unpack the Flash Archive

When the build finishes, BitBake writes a Tegra flash archive under the deploy directory for the machine.

```bash
$ mkdir ~/flash-weston
$ cd ~/flash-weston
$ tar xf ~/tegra-demo-distro/build-weston/tmp/deploy/images/jetson-orin-nano-devkit-nvme/\
demo-image-weston-jetson-orin-nano-devkit-nvme.rootfs.tegraflash-tar.zst
```

The deploy path includes the machine name. The archive name includes both the image name and the machine name.

## 8. Put the Jetson in Recovery Mode

Power off the Jetson. Short J14 pins 9 and 10, then apply power. Those pins are `FC REC` and `GND`. This forces the module into recovery mode.

On the host, verify that USB sees an NVIDIA APX device.

```bash
$ lsusb | grep "NVIDIA Corp. APX"
Bus 007 Device 002: ID 0955:7523 NVIDIA Corp. APX
```

APX is NVIDIA recovery mode. In this mode, the host can push a temporary bootloader over USB and use it to flash the NVMe.

## 9. Flash the Jetson

Run the flash script from the unpacked directory.

```bash
$ sudo ./initrd-flash
```

A successful run ends with `Final status: SUCCESS`.

```text
Final status: SUCCESS
Successfully finished at 2026-05-16T01:12:48-07:00
Host-side log:              log.initrd-flash.2026-05-16-00.32.17
Device-side logs stored in: device-logs-2026-05-16-00.32.17
```

After flashing succeeds, reenable GNOME automount if you disabled it earlier.

```bash
$ gsettings set org.gnome.desktop.media-handling automount true
$ gsettings set org.gnome.desktop.media-handling automount-open true
```

## 10. Verify Boot and DisplayPort

Connect a DisplayPort monitor and boot the Jetson. Weston should start automatically and show a desktop.

If you have a USB to TTL serial adapter, connect it to J14 before booting.

1. Adapter RX goes to J14 TX.
2. Adapter TX goes to J14 RX.
3. Adapter GND goes to J14 GND.

Install and open `minicom` on the host.

```bash
$ sudo apt install minicom
$ sudo minicom -D /dev/ttyUSB0 -b 115200
```

On the serial console, look for `weston.service` starting successfully.

```text
[  OK  ] Started weston.service - Weston Wayland compositor.
```

## 11. Troubleshooting BitBake uid_map Errors

On Ubuntu 24.04, `bitbake demo-image-weston` may fail with this error.

```text
ERROR: PermissionError: [Errno 1] Operation not permitted
...
  File ".../bitbake/lib/bb/utils.py", line 2050, in disable_network
    with open("/proc/self/uid_map", "w") as f:
```

BitBake is trying to create a private user and network namespace for a build task. The kernel allows the namespace creation, then Ubuntu AppArmor blocks the write to `/proc/self/uid_map`.

Check the host settings.

```bash
$ cat /proc/sys/kernel/unprivileged_userns_clone
1
$ cat /proc/sys/kernel/apparmor_restrict_unprivileged_userns
1
```

The first value means unprivileged user namespaces are allowed by the kernel. The second value means Ubuntu AppArmor still restricts them. That combination lets `unshare()` succeed, then blocks the UID map write that BitBake needs next.

The host fix is this sysctl.

```bash
$ sudo sysctl -w kernel.apparmor_restrict_unprivileged_userns=0
```
