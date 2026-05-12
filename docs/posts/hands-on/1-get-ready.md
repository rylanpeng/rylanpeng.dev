---
title: Get Ready!
date: 2026-03-19
categories:
  - Jetson Orin Nano
  - Hands-on
---

# Get Ready!

To start the Jetson Orin Nano journey, the first step is to figure out where to grab the source code and which build tools are needed.

<!-- more -->

---

!!! note "Note"
    These notes are based on the official [NVIDIA Jetson Linux Developer Guide R36.5](https://docs.nvidia.com/jetson/archives/r36.5/DeveloperGuide/index.html).

## Check the L4T Version

Verify the **L4T** (Linux for Tegra) version running on the device. In my case, it is **r36.5**.

The version can be checked using the following command:

```bash hl_lines="2"
rylan-nano@ubuntu:~$ cat /etc/nv_tegra_release 
# R36 (release), REVISION: 5.0, GCID: 43688277, BOARD: generic, EABI: aarch64, DATE: Fri Jan 16 03:50:45 UTC 2026
# KERNEL_VARIANT: oot
TARGET_USERSPACE_LIB_DIR=nvidia
TARGET_USERSPACE_LIB_DIR_PATH=usr/lib/aarch64-linux-gnu/nvidia
```

---

## Download Source Code

### 1. Driver Package (BSP)

1.  Navigate to the [Jetson Linux Archive](https://developer.nvidia.com/embedded/jetson-linux-archive).
2.  Select the version corresponding to the L4T release (e.g., **36.5** - [Direct Link](https://developer.nvidia.com/embedded/jetson-linux-r365)).
3.  Download the **Driver Package (BSP)** from the **DRIVERS** section.

Extract the downloaded file `Jetson_Linux_<version>_aarch64.tbz2` and you'll get the `Linux_for_Tegra` folder.

### 2. BSP Sources

1.  On the same download page, locate the **SOURCES** section.
2.  Download the **Driver Package (BSP) Sources** (`public_sources.tbz2`).


#### Extraction Procedure

Use the following commands to set up the source tree. Replace `<install-path>` with the target directory.

```bash
# Extract public sources
$ tar xf public_sources.tbz2 -C <install-path>/Linux_for_Tegra/..

# Navigate to the source directory
$ cd <install-path>/Linux_for_Tegra/source

# Extract kernel and module component sources
$ tar xf kernel_src.tbz2
$ tar xf kernel_oot_modules_src.tbz2
$ tar xf nvidia_kernel_display_driver_source.tbz2
```

---

## Install Build Dependencies

```bash
$ sudo apt-get update
$ sudo apt-get upgrade -y

# Essential tools and libraries
$ sudo apt install -y build-essential bc flex bison libssl-dev zstd
```