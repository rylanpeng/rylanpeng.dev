---
title: "BitBake config pipeline: the bitbake.conf include chain and the complete namespace"
date: 2026-05-14T20:00:00
categories:
  - Jetson Orin Nano
  - Yocto
  - Deep Dive
---

# BitBake config pipeline: the bitbake.conf include chain and the complete namespace

The previous post, [BitBake config pipeline: the variable store and the layer map](./4-bitbake-config-pipeline-variable-store-and-layer-map.md), ended with `bitbake.conf` located via `BBPATH` and about to be parsed. This post traces what happens inside that parse: how `bitbake.conf` triggers a chain of four more config files, why `local.conf` must come first in that chain, how the machine conf sets `IMAGE_FSTYPES` and `KERNEL_DEVICETREE` through four nested `require` statements, how the distro conf sets `DISTRO_FEATURES` and `INIT_MANAGER`, what the deferred `:append` operator means for parse order, and what the namespace contains when `parseConfigurationFiles()` finally returns.

<!-- more -->

---

!!! note
    BitBake source links point to commit [`941cfe7`](https://git.openembedded.org/bitbake/log/?id=941cfe74c19e01e977101e95b09425d21faf844d) (`yocto-6.0_M3`). openembedded-core source links point to commit [`42fa856a`](https://git.openembedded.org/openembedded-core/log/?id=42fa856a00ac16b2a7a83d7ecfa60a5be192b16c). meta-tegra links point to commit [`2a7fc6f8`](https://github.com/OE4T/meta-tegra/commit/2a7fc6f87d8798355f24eca71c43e49461ac3409). tegra-demo-distro (meta-tegrademo) links point to commit [`d48b1464`](https://github.com/OE4T/tegra-demo-distro/commit/d48b1464061b56b7f679731dbd8b7747c048d1b7).

!!! warning
    This post is my own trace through the source code. I followed the call chain carefully, but my interpretation of what each part does may be incorrect. If you spot an error, please let me know.

---

## Roadmap

```
1.  cookerdata.py:485   parse_config_file("conf/bitbake.conf", data)
                         → openembedded-core/meta/conf/bitbake.conf (~900 lines of defaults)
2.  bitbake.conf:817    include conf/local.conf
                         → MACHINE = "jetson-orin-nano-devkit", DISTRO = "tegrademo"
3.  bitbake.conf:825    include conf/machine/${MACHINE}.conf
                         → jetson-orin-nano-devkit.conf
                              → orin-nano.inc → tegra234.inc → tegra-common.inc
                                 → IMAGE_CLASSES, IMAGE_FSTYPES, KERNEL_DEVICETREE set
4.  bitbake.conf:828    include conf/distro/${DISTRO}.conf
                         → tegrademo.conf → tegrademo.inc
                              → DISTRO_FEATURES = "pam", INIT_MANAGER = "systemd"
5.  bitbake.conf:829    include conf/distro/defaultsetup.conf
                         → init-manager-systemd.inc → DISTRO_FEATURES:append "systemd usrmerge"
6.  cookerdata.py:487   postfiles loop (empty for a plain run) → parseConfigurationFiles() returns
```

---

## `bitbake.conf`: the master config and its include chain

`bitbake.conf` lives in `openembedded-core/meta/conf/` and sets hundreds of default values: compiler flags, sysroot paths, package format defaults, fetch configuration, and more. For this trace, only its tail matters. The tail is a sequence of four `include` and `require` directives that pull in the project-specific configuration.

The parse order is determined strictly by line number. `local.conf` is included at line 817, before the machine conf at line 825 and the distro conf at line 828:

**[`meta/conf/bitbake.conf:817`](https://git.openembedded.org/openembedded-core/tree/meta/conf/bitbake.conf?id=42fa856a00ac16b2a7a83d7ecfa60a5be192b16c#n817)**
```sh
include conf/local.conf
```

**[`meta/conf/bitbake.conf:824–829`](https://git.openembedded.org/openembedded-core/tree/meta/conf/bitbake.conf?id=42fa856a00ac16b2a7a83d7ecfa60a5be192b16c#n824)**
```sh
require ${@"conf/multiconfig/${BB_CURRENT_MC}.conf" if "${BB_CURRENT_MC}" != "" else ""}
include conf/machine/${MACHINE}.conf
include conf/machine-sdk/${SDKMACHINE}.conf
require conf/cve-check-map.conf
include conf/distro/${DISTRO}.conf
include conf/distro/defaultsetup.conf
```

`local.conf` being at line 817, before the machine and distro conf, is not an accident. Line 825 expands `${MACHINE}` and line 828 expands `${DISTRO}`. Those variables must already exist in the datastore before the parser reaches those lines. `local.conf` is what sets them. If the order were reversed, `${MACHINE}` and `${DISTRO}` would expand to empty strings and neither the machine conf nor the distro conf would be found.

---

## `local.conf`: `MACHINE` and `DISTRO` land in the namespace

`local.conf` is the project's hand-edited customization file. For this build:

**[`build-weston/conf/local.conf`](https://github.com/rylanpeng/rylanpeng.dev/blob/main/snippets/local.conf)**

Key lines:
```sh
MACHINE ?= "jetson-orin-nano-devkit"             # line 14  — resolves ${MACHINE} on bitbake.conf:825
DISTRO = "tegrademo"                             # line 255 — resolves ${DISTRO} on bitbake.conf:828
DISTRO_FEATURES:append = " wayland opengl"       # line 258 — adds wayland/opengl to distro features
TNSPEC_BOOTDEV:jetson-orin-nano-devkit = "nvme0n1p1"   # line 261 — NVMe boot target
LICENSE_FLAGS_ACCEPTED = "commercial_tegra-binaries"   # line 264 — NVIDIA BSP license
```

After `local.conf` is parsed, `MACHINE` is `"jetson-orin-nano-devkit"` and `DISTRO` is `"tegrademo"`. The `DISTRO_FEATURES:append` line stores a pending modifier that will be applied at expansion time. That modifier is explained in the `:append` section below.

---

## Machine conf chain: four files deep

`${MACHINE}` is now `"jetson-orin-nano-devkit"`, so BitBake loads:

**[`conf/machine/jetson-orin-nano-devkit.conf`](https://github.com/OE4T/meta-tegra/blob/2a7fc6f87d8798355f24eca71c43e49461ac3409/conf/machine/jetson-orin-nano-devkit.conf)**
```sh
require conf/machine/include/orin-nano.inc    # line 5
require conf/machine/include/devkit-wifi.inc  # line 6
TNSPEC_BOOTDEV_DEFAULT ?= "mmcblk1p1"
```

`require` is like `include` but fails hard if the file is missing. It inlines the next file synchronously, so the parser recurses into the required file before continuing with the current one. The chain continues:

**[`conf/machine/include/orin-nano.inc`](https://github.com/OE4T/meta-tegra/blob/2a7fc6f87d8798355f24eca71c43e49461ac3409/conf/machine/include/orin-nano.inc)**
```sh
TEGRA_BOARDID ?= "3767"
KERNEL_DEVICETREE ?= "tegra234-p3768-0000+p3767-0005-nv-super.dtb"  # line 16
require conf/machine/include/tegra234.inc                             # line 11
```

`KERNEL_DEVICETREE` is the DTB filename that will end up in `flash_weston/`. It is set at this level so that the specific board's device tree is established before the deeper common files run.

**[`conf/machine/include/tegra234.inc`](https://github.com/OE4T/meta-tegra/blob/2a7fc6f87d8798355f24eca71c43e49461ac3409/conf/machine/include/tegra234.inc)**
```sh
SOC_FAMILY = "tegra234"
require conf/machine/include/tegra-common.inc    # line 22
```

**[`conf/machine/include/tegra-common.inc:67–75`](https://github.com/OE4T/meta-tegra/blob/2a7fc6f87d8798355f24eca71c43e49461ac3409/conf/machine/include/tegra-common.inc#L67)**
```sh
IMAGE_CLASSES += "image_types_tegra"                              # line 67
IMAGE_FSTYPES += "tegraflash-tar.zst"                             # line 68
TEGRAFLASH_INITRD_FLASH_IMAGE ?= "tegra-initrd-flash-initramfs"   # line 75
```

These three lines are what makes `flash_weston/` possible. When `image.bbclass` runs later during recipe processing, it reads `IMAGE_CLASSES` and loads `image_types_tegra.bbclass`. That bbclass registers the `do_image_tegraflash_tar` task, which produces the `.tegraflash-tar.zst` artifact. `image.bbclass` also reads `IMAGE_FSTYPES`, sees `"tegraflash-tar.zst"`, and creates the corresponding task with zstd compression. `TEGRAFLASH_INITRD_FLASH_IMAGE` names the pre-built initrd that becomes `initrd-flash` inside the final tarball.

These variables are set in the machine config. The recipe `demo-image-weston.bb` never mentions `tegraflash` at all. It inherits the behavior because `IMAGE_CLASSES` and `IMAGE_FSTYPES` are already in the namespace when the recipe runs.

The full machine conf chain in order:

```
bitbake.conf:825
  → jetson-orin-nano-devkit.conf
      → orin-nano.inc               (KERNEL_DEVICETREE, TEGRA_BOARDID)
          → tegra234.inc            (SOC_FAMILY)
              → tegra-common.inc    (IMAGE_CLASSES, IMAGE_FSTYPES, TEGRAFLASH_INITRD_FLASH_IMAGE)
      → devkit-wifi.inc             (wifi-related settings)
```

---

## Distro conf chain: `DISTRO_FEATURES` and `INIT_MANAGER`

`${DISTRO}` is now `"tegrademo"`, so BitBake loads:

**[`layers/meta-tegrademo/conf/distro/tegrademo.conf:1`](https://github.com/OE4T/tegra-demo-distro/blob/d48b1464061b56b7f679731dbd8b7747c048d1b7/layers/meta-tegrademo/conf/distro/tegrademo.conf#L1)**
```sh
require conf/distro/include/tegrademo.inc
```

**[`layers/meta-tegrademo/conf/distro/include/tegrademo.inc`](https://github.com/OE4T/tegra-demo-distro/blob/d48b1464061b56b7f679731dbd8b7747c048d1b7/layers/meta-tegrademo/conf/distro/include/tegrademo.inc)**

Key lines:
```sh
DISTRO = "tegrademo"                                                 # line 1
TD_DEFAULT_DISTRO_FEATURES = "pam"                                   # line 22
DISTRO_FEATURES ?= "${TD_DEFAULT_DISTRO_FEATURES}"                   # line 23  → "pam"
PREFERRED_PROVIDER_virtual/kernel:tegra = "${TEGRA_DEFAULT_KERNEL}"  # line 28
PACKAGE_CLASSES ?= "package_rpm"                                     # line 35
INIT_MANAGER = "systemd"                                             # line 44
```

After `tegrademo.inc` is parsed, `DISTRO_FEATURES` is `"pam"` and `INIT_MANAGER` is `"systemd"`.

Then `defaultsetup.conf` is included (bitbake.conf line 829):

**[`meta/conf/distro/defaultsetup.conf:20–21`](https://git.openembedded.org/openembedded-core/tree/meta/conf/distro/defaultsetup.conf?id=42fa856a00ac16b2a7a83d7ecfa60a5be192b16c#n20)**
```sh
INIT_MANAGER ??= "systemd"
require conf/distro/include/init-manager-${INIT_MANAGER}.inc
```

`INIT_MANAGER` is already `"systemd"` (set by `tegrademo.inc`), so `??=` is a no-op here. The `require` expands to `conf/distro/include/init-manager-systemd.inc`:

**[`meta/conf/distro/include/init-manager-systemd.inc:1–3`](https://git.openembedded.org/openembedded-core/tree/meta/conf/distro/include/init-manager-systemd.inc?id=42fa856a00ac16b2a7a83d7ecfa60a5be192b16c#n1)**
```sh
DISTRO_FEATURES:append = " systemd usrmerge"
VIRTUAL-RUNTIME_init_manager ??= "systemd"
```

The distro conf chain in order:

```
bitbake.conf:828
  → tegrademo.conf
      → tegrademo.inc               (DISTRO_FEATURES = "pam", INIT_MANAGER = "systemd")
bitbake.conf:829
  → defaultsetup.conf
      → init-manager-systemd.inc    (DISTRO_FEATURES:append = " systemd usrmerge")
```

---

## The `:append` operator and parse order

At this point three different files have written to `DISTRO_FEATURES`:

- `tegrademo.inc:23` sets the base value `"pam"`. This file is parsed at bitbake.conf:828.
- `init-manager-systemd.inc:2` appends `" systemd usrmerge"`. This file is parsed at bitbake.conf:829.
- `local.conf:258` appends `" wayland opengl"`. This file is parsed at bitbake.conf:817, **before** the other two.

`local.conf` was parsed first, yet its `:append` adds `wayland opengl` on top of a base value that did not exist yet at the time `local.conf` ran. This seems like it should cause a problem. It does not, because `:append` is not a live string concatenation.

In BitBake, `:append` stores a **pending modifier**. The modifier is kept separately from the base value and applied only when the variable is expanded, on top of whatever the base value is at that moment. So the final value of `DISTRO_FEATURES` is assembled at expansion time from all three pieces regardless of which piece was written first:

```
base:   "pam"                          (tegrademo.inc:23)
append: " systemd usrmerge"            (init-manager-systemd.inc:2)
append: " wayland opengl"              (local.conf:258)

expanded: "pam systemd usrmerge wayland opengl"
```

Parse order among `:append` modifiers does not affect the final result. All pending appends are applied on top of whatever the base value is at expansion time.

---

## End of `parseConfigurationFiles()`: the `postfiles` loop

After `bitbake.conf` and its entire include chain finish, `parseConfigurationFiles()` closes with the symmetric counterpart to the `prefiles` loop at the top:

**[`lib/bb/cookerdata.py:487–489`](https://git.openembedded.org/bitbake/tree/lib/bb/cookerdata.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n487)**
```python
for p in postfiles:
    data = parse_config_file(p, data)
```

For a plain `bitbake` run, `postfiles` is empty and this loop does nothing. `parseConfigurationFiles()` returns the completed `DataSmart`.

---

## The complete namespace

The namespace now contains, among thousands of other variables:

| Variable | Final value | Where it was set |
|----------|-------------|-----------------|
| `MACHINE` | `"jetson-orin-nano-devkit"` | local.conf:14 |
| `DISTRO` | `"tegrademo"` | tegrademo.inc:1 |
| `DISTRO_FEATURES` | `"pam systemd usrmerge wayland opengl"` | tegrademo.inc:23; init-manager-systemd.inc:2; local.conf:258 |
| `IMAGE_CLASSES` | includes `"image_types_tegra"` | tegra-common.inc:67 |
| `IMAGE_FSTYPES` | includes `"tegraflash-tar.zst"` | tegra-common.inc:68 |
| `INIT_MANAGER` | `"systemd"` | tegrademo.inc:44 |
| `KERNEL_DEVICETREE` | `"tegra234-p3768-0000+p3767-0005-nv-super.dtb"` | orin-nano.inc:16 |
| `LICENSE_FLAGS_ACCEPTED` | `"commercial_tegra-binaries"` | local.conf:264 |
| `TNSPEC_BOOTDEV` | `"nvme0n1p1"` | local.conf:261 |
| `TEGRAFLASH_INITRD_FLASH_IMAGE` | `"tegra-initrd-flash-initramfs"` | tegra-common.inc:75 |

---

## Execution unwinds back to the UI

`parseConfigurationFiles()` returns the completed `DataSmart`. Execution unwinds through the call chain exactly as it came in:

```
parseConfigurationFiles()      ← returns completed DataSmart
  → parseBaseConfiguration()   ← cookerdata.py:274: self.data = returned DataSmart
  → initConfigurationData()    ← cooker.py:301: self.data = self.databuilder.data
  → init_configdata()          ← cooker.py:199: hasattr(self, "data") is now True
  → command.runCommand()       ← command.py:65: resumes the original getVariable command
  → process_server.main()      ← sends reply over socket back to UI client
```

All of this ran synchronously inside the single `runCommand("getVariable", "BBINCLUDELOGS")` call. The UI client was blocked waiting for the reply. It receives the value of `BBINCLUDELOGS` and continues with the rest of the handshake: `setEventMask`, `getVariable BB_DEFAULT_TASK`, `setConfig cmd build`, and finally `buildTargets ["demo-image-weston"] "build"`. Each of those subsequent commands also calls `init_configdata()`, but `self.data` now exists, so it returns immediately as a no-op.

The config pipeline is complete. The namespace is fully populated and frozen. Everything from here is recipe parsing.
