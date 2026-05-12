---
title: "Yocto Basics Part 3: Kernel, SystemD & Tooling (Notes from Tech-A-Byte)"
date: 2026-05-11T23:00:00
categories:
  - Yocto
  - Lecture Notes
---

# Yocto Basics Part 3: Kernel, SystemD & Tooling (Notes from Tech-A-Byte)

Part 3 covers the more advanced territory: writing SystemD service recipes, configuring the kernel via menuconfig, adding kernel modules (both out-of-tree and in-tree), reusing logic with `.bbclass`, iterating with `devtool`, and customizing your distribution.

<!-- more -->

---

!!! note "Note"
    These notes follow the [Tech-A-Byte Yocto lecture series](https://youtube.com/playlist?list=PLwqS94HTEwpQmgL1UsSwNk_2tQdzq3eVJ) ([GitHub](https://github.com/Munawar-git/YoctoTutorials/tree/master)) using BeagleBone Yocto as the target board. Replace `<your-path>` with your actual workspace root.

## SystemD

Yocto supports three init managers: **SystemD** (parallel startup, feature-rich), **SysVinit** (Yocto's default, sequential), and **BusyBox** (minimal).

To switch to SystemD, add the following to `local.conf`:

```conf
DISTRO_FEATURES:append = " systemd usrmerge"
VIRTUAL-RUNTIME_init_manager = "systemd"
DISTRO_FEATURES_BACKFILL_CONSIDERED += "sysvinit"
VIRTUAL-RUNTIME_initscripts = "systemd-compat-units"
```

- `DISTRO_FEATURES:append = " systemd"`: declares that the distro supports systemd. Does not by itself select it as the init system.
- `VIRTUAL-RUNTIME_init_manager = "systemd"`: explicitly selects systemd as the init manager.
- `DISTRO_FEATURES_BACKFILL_CONSIDERED += "sysvinit"`: ensures sysvinit compatibility is considered. Useful for software that still relies on SysVinit-style init scripts.
- `VIRTUAL-RUNTIME_initscripts = "systemd-compat-units"`: uses systemd compatibility units to manage SysVinit-style services under systemd.

On the embedded device, verify systemd is running:

```bash
$ ps ax
$ systemctl
$ dmesg
```

---

## SystemD Service Recipe

A systemd service is **a unit of work that systemd can start, stop, and manage**. It's defined by a `.service` unit file.

### Writing a Recipe for a SystemD Service

1. Create the script or binary the service will run.
2. Create a `.service` unit file defining the service's behavior.
3. Place both in the recipe's `files/` directory.
4. Use `do_install` to copy them to the correct locations in the rootfs.
5. Declare dependencies and metadata.

### Where to place the service file

Use `do_install` with the correct FHS path. Both locations below work:

- `${sysconfdir}/systemd/system` (`/etc/systemd/system`)
- `${systemd_unitdir}/system` (`/lib/systemd/system`)

### How to enable the service by default

```conf
SYSTEMD_AUTO_ENABLE = "enable"
SYSTEMD_SERVICE:${PN} = "sysd.service"
```

### Full Example

```
<your-path>/poky/meta-mylayer/recipes-example/systemd-example/
├── files/
│   ├── sysd.service
│   └── sysd.sh
└── systemd-example.bb
```

`sysd.sh`:
```bash
#!/bin/bash
while true; do
    echo "Hello, world!"
    sleep 1
done
```

`sysd.service`:
```conf
[Unit]
Description=Continuous Hello World Service

[Service]
ExecStart=/bin/bash /usr/bin/sysd.sh

[Install]
WantedBy=multi-user.target
```

`systemd-example.bb`:
```conf
SUMMARY = "SystemD Hello World service example"
DESCRIPTION = "A simple looping service managed by systemd"
LICENSE = "MIT"

LIC_FILES_CHKSUM = "file://${COREBASE}/meta/COPYING.MIT;md5=3da9cfbcb788c80a0384361b4de20420"

SRC_URI = "file://sysd.sh \
           file://sysd.service \
           "

inherit systemd

S = "${WORKDIR}"

RDEPENDS:${PN} = "bash"

SYSTEMD_AUTO_ENABLE = "enable"
SYSTEMD_SERVICE:${PN} = "sysd.service"

do_install() {
    install -d ${D}${bindir}
    install -m 0755 ${S}/sysd.sh ${D}${bindir}

    # sysconfdir = /etc
    install -d ${D}/${sysconfdir}/systemd/system
    install -m 0644 ${S}/sysd.service ${D}/${sysconfdir}/systemd/system
}
```

In `local.conf`:
```conf
IMAGE_INSTALL:append = " systemd-example"
```

### Basic SystemD Commands

```bash
$ systemctl status sysd          # check service status
$ systemctl start sysd           # start
$ systemctl stop sysd            # stop
$ systemctl restart sysd         # restart
$ systemctl enable sysd          # enable at boot
$ systemctl disable sysd         # disable at boot
$ systemctl daemon-reload        # reload after editing unit files
```

---

## Menuconfig

By default, BeagleBone Yocto does not expose GPIO via sysfs. To enable it, reconfigure the kernel.

On the embedded device first, confirm GPIO is missing:

```bash
$ ls /sys/class
# no gpio entry
```

On the host, open the kernel config UI:

```bash
$ bitbake -c menuconfig virtual/kernel
```

Navigate to:

1. **General setup** → Enable **"Configure standard kernel features (expert users)"**
2. **Device Drivers** → **GPIO Support** → Enable **"/sys/class/gpio/... (sysfs interface)"**

Then rebuild:

```bash
$ bitbake core-image-full-cmdline
```

On the embedded device, verify:

```bash
$ ls /sys/class
# gpio is now visible
```

---

**Q:** What does `bitbake -c menuconfig virtual/kernel` actually do? Does it build the image?

It does **not** build the image. It runs only the `do_menuconfig` task for `virtual/kernel`. It fetches and unpacks the kernel source if needed, then opens the ncurses kernel config UI. When you save and exit, the updated `.config` is stored in the kernel's build directory. You still need `bitbake core-image-full-cmdline` to rebuild the image.

`virtual/kernel` is a virtual target that BitBake resolves to the actual kernel recipe (e.g. `linux-yocto`) via `PREFERRED_PROVIDER_virtual/kernel` in your config.

---

## savedefconfig

To avoid running menuconfig on every clean build, save the kernel config as a `defconfig` file inside a `.bbappend` so it persists.

**Steps:**

1. Create a kernel `.bbappend`.
2. Run menuconfig and make your changes.
3. Save with `savedefconfig`.
4. Copy the generated file into the recipe folder.
5. Rebuild the image.

Recipe structure:

```
<your-path>/poky/meta-mylayer/recipes-kernel/linux/
├── linux-yocto/
│   └── defconfig
└── linux-yocto_5.15.bbappend
```

`linux-yocto_5.15.bbappend`:
```conf
FILESEXTRAPATHS:prepend := "${THISDIR}/${PN}:"
SRC_URI += "file://defconfig"
KCONFIG_MODE = "alldefconfig"
```

```bash
$ bitbake -c menuconfig virtual/kernel
# enable GPIO sysfs as before

$ bitbake -c savedefconfig virtual/kernel

$ cd <your-path>/sources/tmp/work/<arch>/linux-yocto/5.15.xxx/linux-beaglebone_yocto-standard-build
$ pwd
# <kernel-build-path>

$ cd <your-path>/poky/meta-mylayer/recipes-kernel/linux/linux-yocto
$ cp <kernel-build-path>/defconfig .
```

Verify the config is picked up on a clean build:

```bash
$ bitbake -c cleansstate virtual/kernel
$ bitbake -c menuconfig virtual/kernel
# GPIO sysfs should already be enabled
```

---

## diffconfig and fragment .cfg

Instead of replacing the entire `defconfig`, save only the delta as a `.cfg` fragment. This keeps the base config untouched and layers your changes on top.

```bash
$ bitbake -c diffconfig virtual/kernel

$ cd <your-path>/sources/tmp/work/<arch>/linux-yocto/5.15.xxx/
$ ls
# ... fragment.cfg ...
$ pwd
# <fragment-path>

$ cd <your-path>/poky/meta-mylayer/recipes-kernel/linux/linux-yocto
$ cp <fragment-path>/fragment.cfg gpio-sysfs.cfg
```

In `linux-yocto_5.15.bbappend`:
```conf
FILESEXTRAPATHS:prepend := "${THISDIR}/${PN}:"
SRC_URI += "file://gpio-sysfs.cfg"
```

---

## Out-of-Tree Module

An out-of-tree module lives entirely in your layer. `module.bbclass` handles cross-compiling it against the kernel headers externally.

```
<your-path>/poky/meta-mylayer/recipes-example/hello-mod/
├── files/
│   ├── COPYING
│   ├── hello.c
│   └── Makefile
└── hello-mod_0.1.bb
```

`hello.c`:
```c
#include <linux/module.h>
static int __init hello_init(void) { pr_info("Hello World!\n"); return 0; }
static void __exit hello_exit(void) { pr_info("Goodbye!\n"); }

module_init(hello_init);
module_exit(hello_exit);
MODULE_LICENSE("GPL");
```

`Makefile`:
```makefile
obj-m := hello.o

SRC := $(shell pwd)

all:
	$(MAKE) -C $(KERNEL_SRC) M=$(SRC)

modules_install:
	$(MAKE) -C $(KERNEL_SRC) M=$(SRC) modules_install

clean:
	rm -f *.o *~ core .depend .*.cmd *.ko *.mod.c
	rm -f Module.markers Module.symvers modules.order
	rm -rf .tmp_versions Modules.symvers
```

`hello-mod_0.1.bb`:
```conf
SUMMARY = "Hello World kernel module"
DESCRIPTION = "Out-of-tree kernel module example"
LICENSE = "GPL-2.0-only"
LIC_FILES_CHKSUM = "..."

inherit module

SRC_URI = "file://Makefile \
           file://hello.c \
           file://COPYING \
          "
S = "${WORKDIR}"

RPROVIDES:${PN} += "kernel-module-hello"
```

Build and verify on the embedded device:

```bash
$ bitbake core-image-full-cmdline
```

```bash
$ ls /lib/modules/5.15.54-yocto-standard/extra
# hello.ko

$ modprobe hello
$ dmesg | tail
$ lsmod

$ rmmod hello
$ dmesg | tail
$ lsmod
```

---

## In-Tree Module (devshell + git patch)

An in-tree module is inserted directly into the kernel source tree. It appears as a first-class kernel option in menuconfig and is compiled as part of the full kernel build.

### Step 1: Get kernel source via devshell

```bash
$ bitbake -c devshell virtual/kernel
$ cd <your-path>/sources/tmp/work-shared/beaglebone-yocto/kernel-source
$ ls
```

### Step 2: Initialize git

```bash
$ git init
$ git add *
$ git commit -m "Initialize"
```

### Step 3: Create the driver folder

```bash
$ cd drivers/char
$ mkdir tabdriver && cd tabdriver
```

### Step 4: Create Kconfig and Makefile

```bash
$ touch Kconfig && touch Makefile
```

`Kconfig`:
```conf
config TAB_DRIVER
    tristate "TAB Hello World Driver"
    help
        Tech A Byte Hello World Driver
```

`Makefile`:
```makefile
obj-$(CONFIG_TAB_DRIVER) := tabdriver.o
```

### Step 5: Copy the driver source

```bash
$ cp <source>/tab-module.c .
$ mv tab-module.c tabdriver.c
```

### Step 6: Hook into the parent Kconfig and Makefile

In `drivers/char/Kconfig`, add:
```
source "drivers/char/tabdriver/Kconfig"
```

In `drivers/char/Makefile`, add:
```makefile
obj-$(CONFIG_TAB_DRIVER) := tabdriver/
```

### Step 7: Create a patch with git

```bash
$ cd <your-path>/sources/tmp/work-shared/beaglebone-yocto/kernel-source
$ git add drivers/ && git commit -m "tab char driver"
$ git format-patch HEAD~1
# generates: 0001-tab-char-driver.patch

$ pwd
# <kernel-source-path>

$ cd <your-path>/poky/meta-mylayer/recipes-kernel/linux/linux-yocto
$ cp <kernel-source-path>/0001-tab-char-driver.patch .
```

In `linux-yocto_5.15.bbappend`:
```conf
SRC_URI += "file://0001-tab-char-driver.patch"
```

```bash
$ bitbake -c menuconfig virtual/kernel
# Device Drivers -> Character devices -> TAB Hello World Driver is now visible
```

---

### Summary: Kernel Config & Module Methods

| # | Method | Type | Verification |
|---|--------|------|-------------|
| 1 | menuconfig only | Temporary (this build only) | `ls /sys/class` shows `gpio` |
| 2 | menuconfig + savedefconfig → `defconfig` in recipe | Permanent full config | `ls /sys/class` shows `gpio` |
| 3 | menuconfig + diffconfig → `fragment.cfg` in recipe | Permanent config delta | `ls /sys/class` shows `gpio` |
| 4 | New recipe + `inherit module` | Out-of-Tree Module | `ls /lib/modules/.../extra` shows `.ko` |
| 5 | devshell + Kconfig/Makefile + git patch | In-Tree Module | Driver appears in menuconfig; module in kernel |

**Out-of-Tree (method 4):** driver source lives in your layer's `files/`. `module.bbclass` cross-compiles it against kernel headers externally. Result: `.ko` in `/lib/modules/.../extra/`, loaded via `modprobe`.

**In-Tree (method 5):** you literally insert code into the kernel source tree. The driver becomes a first-class kernel option visible in menuconfig under the right category.

---

## bbclass

`.bbclass` files are located in a `classes/` directory within a layer. Any `.bb` or `.bbappend` can pull in a class with `inherit classname`.

```bash
$ cd <your-path>/poky/meta/classes
$ code systemd.bbclass
# contains SYSTEMD_AUTO_ENABLE, SYSTEMD_SERVICE, etc. (the same variables we used earlier)
```

---

### Q&A: .bbclass vs .inc

**Q:** What is the difference between `.bbclass` and `.inc`? What about `RDEPENDS`?

- **`.inc`**: a partial `.bb` file written in the same BitBake language. Used to share **variable definitions and configuration snippets** across multiple recipes. Included with `require file.inc` or `include file.inc`. Think of it as copy-pasting a config block in.

- **`.bbclass`**: defines **reusable behaviors and tasks**. Same base language as `.bb`, but with extra capabilities: `addtask`, `EXPORT_FUNCTIONS`, Python task support. Pulled in with `inherit classname`. No path or extension is needed. BitBake searches `classes/` dirs across all configured layers automatically. The key difference from `.inc`: classes provide **behaviors** (task implementations). `.inc` provides **data/variable snippets**.

- **`RDEPENDS`**: completely unrelated to file inclusion. It's a runtime package dependency declaration: "when this package is installed on the target, also install these other packages."

---

**Q:** If I put `ZZZZZZ.bbclass` in a `classes/` folder of one layer, can recipes in other layers use `inherit ZZZZZZ`?

Yes. BitBake searches `classes/` directories across **all** layers listed in `BBLAYERS`. Any `.bbclass` in any configured layer is globally available: just use `inherit ZZZZZZ`.

Example `ZZZZZZ.bbclass`:
```conf
do_fetch_info() {
    bbplain "****************************"
    bbplain "*   Fetching ${PN}_${PV}   *"
    bbplain "****************************"
}
addtask do_fetch_info before do_fetch
```

In `bbb-example.bb`:
```conf
inherit ZZZZZZ
```

Classes can also `inherit` other classes. Nested inheritance works.

---

### File Type Reference

| File type | How it includes others | Can be used by |
|-----------|----------------------|--------------------|
| `.bb` | `require`/`include` + `inherit` | (end of chain) |
| `.bbappend` | `require`/`include` + `inherit` | (auto-matched by filename) |
| `.bbclass` | `inherit` (nested classes) | `.bb`, `.bbappend`, `.bbclass` via `inherit` |
| `.inc` | `require`/`include` | `.bb`, `.bbappend`, `.conf` via `require`/`include` |
| `.conf` | `require`/`include` | Other `.conf` files |

Rules:
- `require path/to/file`: hard include (error if missing)
- `include path/to/file`: soft include (silently skips if missing)
- `inherit classname`: pulls in `classname.bbclass`. No path or extension needed. BitBake searches all `classes/` dirs.
- `.conf` files **cannot** use `inherit`. That applies only to recipes and classes.

---

## Searching Recipes

Recipes included in `BBLAYERS`:

```bash
$ bitbake-layers show-recipes
$ bitbake-layers show-recipes | grep -A 1 "python.*:"
```

All `.bb` files on disk (regardless of `BBLAYERS`):

```bash
$ cd <your-path>/poky
$ find -iname "*.bb" | grep hello
$ find -iname "*.bbappend"
$ find -iname "*.inc"
```

---

## devtool

`devtool` accelerates recipe development with a `workspace/` sandbox. It has three main subcommands: `add`, `modify`, and `upgrade`.

!!! note
    Unlike regular Yocto work (where custom layers live **outside** `build/`), devtool works **inside** `<your-path>/poky/build/`.

```bash
$ devtool -h
$ devtool add -h
```

### devtool Workflow Overview

```
              "Target"
                 ^
 undeploy-target | deploy-target
                 |
   finish        v          add / upgrade
"Layers" <---> "Workspace" <--- "Source"
                  build
                  modify
```

---

### add

Pull in a new external source and generate a recipe scaffold:

```bash
$ cd <your-path>/poky/build
$ devtool add hello-dev https://github.com/Munawar-git/bbb-tutorial-code.git
$ ls workspace/
# appends   conf   README   recipes   sources
$ code workspace/recipes/hello-dev/hello-dev_git.bb
$ ls workspace/sources/hello-dev
```

Edit the recipe, then build:

```bash
$ devtool build hello-dev
# binary appears in workspace/sources/hello-dev/
```

Another example, factorial:

```bash
$ devtool add factorial https://github.com/Munawar-git/factorial.git --srcbranch=main
# add <your-path>/poky/build/workspace to BBLAYERS in bblayers.conf
```

Adjust `factorial_git.bb`:
```conf
EXTRA_OEMAKE = "PREFIX=${D}${bindir} CC='${CC}' CFLAGS='${CFLAGS}'"
```

Build and deploy directly to the running device:

```bash
$ devtool build factorial
$ devtool deploy-target factorial root@192.168.7.2
```

On the embedded device:
```bash
$ which factorial
# /usr/bin/factorial
$ factorial
# Enter a number ...
```

```bash
$ devtool undeploy-target factorial root@192.168.7.2
```

Before running the full `bitbake factorial`, add one more flag for strict cross-linker compatibility:
```conf
EXTRA_OEMAKE = "PREFIX=${D}${bindir} CC='${CC}' CFLAGS='${CFLAGS} -Wl,--hash-style=gnu'"
```

---

**Q:** `deploy-target` already puts the binary on the device. What's the difference between that and `devtool finish`?

`deploy-target` copies the built binary to the running device over SSH for quick iteration testing. Nothing in your layers changes. `devtool finish` is completely different: it moves the recipe out of `workspace/` and into a proper meta-layer permanently. After `finish`, the workspace no longer has that recipe, and you can remove it from `BBLAYERS` and delete `workspace/` entirely. They serve separate purposes: `deploy-target` = quick test on device, `finish` = make it permanent.

---

**Q:** Why does `bitbake factorial` need `-Wl,--hash-style=gnu` when `devtool build` didn't?

`devtool build` is optimized for fast iteration and may be more permissive about cross-compilation flags. `bitbake factorial` goes through the full Yocto pipeline: strict cross-compilation toolchain and all QA checks. Some target architectures, especially older ARM systems, require `--hash-style=gnu` to produce ELF binaries compatible with the target's dynamic linker. `devtool build`'s toolchain may not enforce this.

---

### modify

Pull an existing recipe into workspace for editing, then push changes back:

```bash
$ devtool modify factorial
# Recipe factorial now set up to build from workspace/sources/factorial

$ cd <your-path>/poky/build/workspace/sources/factorial
# make source edits

$ devtool build factorial
$ devtool deploy-target factorial root@192.168.7.2
# test on device
$ devtool undeploy-target factorial root@192.168.7.2

$ git add factorial.c && git commit -m "Modify error messages"
$ devtool finish -f factorial ../../../meta-append-layer/
```

---

**Q:** In the video, `devtool finish` kept failing with "source tree is not clean". What's actually happening? Which git repo does `git add/commit` apply to?

Full picture:

1. `factorial_git.bb` in your layer has `SRC_URI = "git://github.com/.../factorial.git"`.
2. `devtool modify factorial` clones that repo into `workspace/sources/factorial/`. This is a local clone of the **factorial project's git history**, not poky's git.
3. `git add` and `git commit` in `workspace/sources/factorial/` commit to that local factorial clone.
4. `devtool finish` checks that this local clone is clean. The problem: `do_compile` placed a compiled `factorial` binary inside `workspace/sources/factorial/` (since `S` points there). Git sees this untracked binary and reports a dirty tree.
5. Fix without `-f`: `rm factorial` to remove the compiled binary, then `devtool finish` works cleanly.

The `-f` flag bypasses this check. It is risky if you have unsaved source changes.

---

### upgrade

Pull an updated version of a recipe into workspace:

```bash
$ devtool upgrade factorial -S 27134a05xxxxx51e0 -V 2.0.0
$ code <your-path>/poky/build/workspace/recipes/factorial/factorial_git.bb

$ devtool rename factorial -V 2.0.0
$ ls workspace/recipes/factorial/
# factorial_2.0.0.bb

$ devtool build factorial
$ devtool deploy-target factorial root@192.168.7.2
# test on device
$ devtool undeploy-target factorial root@192.168.7.2

# remove the compiled binary so the tree is clean
$ cd <your-path>/poky/build/workspace/sources/factorial
$ rm factorial

$ cd <your-path>/poky/build/workspace
$ devtool finish factorial ../../../meta-tab-layer/

$ rm -rf sources/
$ cd <your-path>/poky/build
$ bitbake factorial
```

---

## DISTRO

A custom distro config lives in a layer's `conf/distro/` folder. It sets distribution-wide settings like `DISTRO_FEATURES`, `VIRTUAL-RUNTIME_xxx`, and which packages are skipped.

```bash
$ cd <your-path>/poky/meta-poky/conf/distro
$ ls
# include   poky-altcfg.conf   poky-bleeding.conf   poky.conf   poky-tiny.conf

$ code poky.conf
$ code poky-tiny.conf
```

Create a custom distro:

```bash
$ touch <your-path>/poky/meta-tab-layer/conf/distro/tab-distro.conf
```

`tab-distro.conf`:
```conf
require conf/distro/poky.conf
DISTRO = "tab-distro"
DISTRO_NAME = "Tech-A-Byte Distro, based upon Poky Distro"
```

In `local.conf`:
```conf
DISTRO = "tab-distro"
```

```bash
$ bitbake core-image-full-cmdline
```

On the embedded device:
```bash
$ cat /etc/os-release
# ID=tab-distro
# NAME=...
```

---

### Q&A: .conf vs .inc vs .bbclass

**Q:** Is `distro.conf` like `.bbclass` (or `.inc`) for `.bb`?

More like `.conf` using `require`, similar to how `.inc` works. It is **not** like `.bbclass`: it doesn't define tasks or behaviors. It just sets configuration variables. Think of it this way:

- `.conf` = project/distro-level identity (who you are, what features your distro has)
- `.inc` = shared recipe data snippets (reusable variable blocks)
- `.bbclass` = reusable task logic (how to build a certain type of thing)

---

**Q:** What's the difference between `IMAGE_INSTALL` and `CORE_IMAGE_EXTRA_INSTALL`?

`IMAGE_INSTALL` is the definitive package list for the image. You can set or append to it anywhere. `CORE_IMAGE_EXTRA_INSTALL` is a convenience variable that only exists when you `inherit core-image`. Whatever you put there automatically gets appended to `IMAGE_INSTALL`.

The intent is clear separation: the image recipe defines `IMAGE_INSTALL` for its required base packages. `CORE_IMAGE_EXTRA_INSTALL` is the slot for user additions. Using `CORE_IMAGE_EXTRA_INSTALL += "nano"` in `local.conf` signals "user addition" rather than "base image requirement."

---

## Build Mode (dev vs release)

Use BitBake's inline Python (`bb.utils.contains`) to conditionally include packages or config fragments based on a custom variable.

In `local.conf`:
```conf
BUILD_MODE = "development"
CORE_IMAGE_EXTRA_INSTALL += "${@bb.utils.contains('BUILD_MODE', 'development', 'nano', '', d)}"
```

In `linux-yocto_5.15.bbappend`:
```conf
FILESEXTRAPATHS:prepend := "${THISDIR}/${PN}:"
SRC_URI += "file://0001-tab-char-driver.patch"
SRC_URI:append = " ${@bb.utils.contains('BUILD_MODE', 'development', 'file://gpio-sysfs.cfg', '', d)}"
```

```bash
$ bitbake -c menuconfig virtual/kernel
# kernel config differs depending on whether BUILD_MODE is 'development' or anything else
```

---

## MACHINE

The machine config defines the target hardware: CPU architecture, kernel provider, and bootloader settings.

```bash
$ cd <your-path>/poky
$ find . -name "*.conf" | grep "conf/machine/"
$ code ./meta-yocto-bsp/conf/machine/beaglebone-yocto.conf
```

---

## Building for a New Board: Raspberry Pi Example

To build for a different target board, find its BSP layer on the OpenEmbedded Layer Index, clone it, and add it to `BBLAYERS`.

1. Go to the [OpenEmbedded Layer Index](https://layers.openembedded.org/layerindex/branch/master/layers/).
2. Click **Machine** and search for "raspberry".
3. Click `meta-raspberrypi` and copy the git URL.

```bash
$ cd <your-path>/poky
$ git clone -b kirkstone git://git.yoctoproject.org/meta-raspberrypi
$ cd meta-raspberrypi
$ find -iname "*.conf" | grep "machine"

$ cd <your-path>/poky/build
$ bitbake-layers add-layer ../meta-raspberrypi/
```

In `local.conf`:
```conf
MACHINE = "raspberrypi3-64"
```

```bash
$ bitbake core-image-full-cmdline
```

---

**Q:** In the video, a TTL serial connection is used to connect to the Raspberry Pi. Is it just for boot messages, or does it give full shell access?

Full shell access. The serial TTL connection at 115200 baud via `picocom` gives you all boot output AND the login prompt once boot completes. This is the standard way to interact with embedded Linux devices that have no HDMI display. The kernel and init system both write to this serial console, and once boot finishes you get a full shell.
