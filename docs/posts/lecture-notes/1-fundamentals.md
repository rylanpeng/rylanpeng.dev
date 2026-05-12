---
title: "Yocto Basics Part 1: Fundamentals (Notes from Tech-A-Byte)"
date: 2026-05-11T21:00:00
categories:
  - Yocto
  - Lecture Notes
---

# Yocto Basics Part 1: Fundamentals (Notes from Tech-A-Byte)

A running set of notes from working through the Tech-A-Byte Yocto lecture series. Part 1 covers the core configuration files, the layer/recipe hierarchy, key directory variables, and BitBake assignment syntax, which is the foundation everything else builds on.

<!-- more -->

---

!!! note "Note"
    These notes follow the [Tech-A-Byte Yocto lecture series](https://youtube.com/playlist?list=PLwqS94HTEwpQmgL1UsSwNk_2tQdzq3eVJ) ([GitHub](https://github.com/Munawar-git/YoctoTutorials/tree/master)) using BeagleBone Yocto as the target board. Replace `<your-path>` throughout with your actual workspace root (the directory that contains both `poky/` and `sources/`).

## local.conf Setup

`local.conf` lives at `<your-path>/poky/build/conf/local.conf`. It controls build-wide settings, most importantly where downloads, sstate cache, and build artifacts are stored.

```conf
SOURCES = "<your-path>/sources"   # must be an absolute path
DL_DIR ?= "${SOURCES}/downloads"
SSTATE_DIR ?= "${SOURCES}/sstate-cache"
TMPDIR = "${SOURCES}/tmp"

RM_OLD_IMAGE = "1"
INHERIT += "rm_work"
```

Build the image:

```bash
$ bitbake core-image-full-cmdline
```

Output lands in `<your-path>/sources/tmp/deploy/images/beaglebone-yocto`. Look for the `.wic` image file:

```bash
$ cd <your-path>/sources/tmp/deploy/images/beaglebone-yocto
$ ll | grep "cmdline" | grep ".wic"
```

To connect to the board via serial (once flashed):

```bash
$ dmesg | grep ftdi
$ ll /dev/ttyUSB0
$ sudo picocom -b 115200 /dev/ttyUSB0
```

Once logged in, verify the running system:

```bash
$ uname -r
$ uname -a
$ lsblk
```

**Key variables:**

**`DL_DIR`**: where all source tarballs and git clones are downloaded.  
**`SSTATE_DIR`**: shared state cache. Yocto reuses these on subsequent builds to skip recompiling unchanged tasks.  
**`TMPDIR`**: staging area for the current build. `INHERIT += "rm_work"` cleans per-recipe work directories after packaging, keeping disk usage under control.

`USER_CLASSES ?= "buildstats"` enables per-task performance statistics:

```bash
$ cat <your-path>/sources/tmp/buildstats/<recipe>/<task>/<timestamp>
```

---

## bblayers.conf

`bblayers.conf` lives at `<your-path>/poky/build/conf/bblayers.conf`. It lists which layers are active in the build.

```conf
POKY_BBLAYERS_CONF_VERSION = "2"

BBPATH = "${TOPDIR}"
BBFILES ?= ""

BBLAYERS ?= " \
  <your-path>/poky/meta \
  <your-path>/poky/meta-poky \
  <your-path>/poky/meta-yocto-bsp \
  "
```

The hierarchy is: **`poky` → `layer` → `recipe` → `.bb`**

Each layer contains recipes. Each recipe has at least one `.bb` file. To browse an existing one:

```bash
$ code <your-path>/poky/meta/recipes-extended/minicom/minicom_2.8.bb
```

To manage layers from within `poky/build/`:

```bash
$ bitbake-layers show-layers
$ bitbake-layers add-layer ../meta-skeleton/
$ bitbake-layers remove-layer meta-skeleton
```

---

## Adding Packages

To see what packages are available across all active layers:

```bash
$ bitbake-layers show-recipes
$ bitbake-layers show-recipes python3
$ bitbake-layers show-recipes git
```

!!! note
    `bitbake-layers show-recipes` only searches layers listed in `BBLAYERS`. To search all `.bb` files on disk regardless of active layers:
    ```bash
    $ find <your-path>/poky -iname "*.bb" | grep hello
    ```

To add packages to the image, append to `IMAGE_INSTALL` in `local.conf`:

```conf
IMAGE_INSTALL:append = " python3"
IMAGE_INSTALL:append = " git"
```

---

## Adding Layers

Create a custom layer and register it:

```bash
$ cd <your-path>/poky/build
$ bitbake-layers create-layer ../meta-mylayer
$ ls <your-path>/poky/meta-mylayer
# conf    COPYING.MIT     README  recipes-example
```

Inspect the generated layer config, register it, and verify:

```bash
$ code <your-path>/poky/meta-mylayer/conf/layer.conf
$ bitbake-layers add-layer ../meta-mylayer
$ bitbake-layers show-layers
```

Test with the auto-generated example recipe:

```bash
$ bitbake example
```

This builds `<your-path>/poky/meta-mylayer/recipes-example/example/example.bb`.

---

## Yocto Variables: PN, PV, PR, WORKDIR, S, B, D

Recipe filenames follow the format **`PN_PV_PR.bb`** (Package Name, Version, Revision). Yocto derives several important directory variables from these:

| Variable | Default Value |
|----------|---------------|
| `WORKDIR` | `TMPDIR/<machine>/<PN>/<PV>-<PR>` |
| `S` (Source) | `WORKDIR/<PN>-<PV>` |
| `B` (Build) | `S` |
| `D` (Destination) | `WORKDIR/image` |

Create `example1_0.1_r1.bb` in `<your-path>/poky/meta-mylayer/recipes-example/examples/` and verify:

```bash
$ cd <your-path>/poky/build
$ bitbake example1

$ bitbake -e example1 | grep ^PN=
# PN="example1"

$ bitbake -e example1 | grep ^WORKDIR=
# WORKDIR="<your-path>/sources/tmp/work/<arch>/example1/0.1-r1"

$ bitbake -e example1 | grep ^S=
# S="<your-path>/sources/tmp/work/<arch>/example1/0.1-r1/example1-0.1"

$ bitbake -e example1 | grep ^D=
# D="<your-path>/sources/tmp/work/<arch>/example1/0.1-r1/image"
```

---

### Q&A: WORKDIR / S / B / D

**Q:** When a `.bb` file sets `S = "${WORKDIR}/build"`, how many variables change? What about `D` and `B`? Are these variables private to the recipe?

Setting `S = "${WORKDIR}/build"` changes `S` explicitly. Because `B` defaults to `${S}`, `B` changes automatically too. `D` is independent and stays at `${WORKDIR}/image` unless explicitly overridden.

These variables are **private to the recipe**. Other recipes cannot reach inside them. To share output with dependent recipes, use `${D}` and let Yocto's standard pipeline handle the rest:

- `do_install` → copies files into `${D}`
- `do_populate_sysroot` → stages headers/libs from `${D}` into a shared sysroot, available to any recipe that `DEPENDS` on this one
- `do_package` → turns `${D}` contents into installable packages for the final image

---

**Q:** What happens if `do_install` writes files to a path outside `${D}`?

`do_package` only looks inside `${D}`. Files placed elsewhere are invisible to packaging. Yocto generates an empty package and emits a QA warning: `WARNING: helloworld is empty and ALLOW_EMPTY is not set`. Those files never reach the final image.

---

**Q:** What if I set `D = "${WORKDIR}/apple"`?

It technically works, since Yocto reads the variable rather than a hardcoded path. But it's strongly discouraged. Some third-party classes hardcode the word `image` instead of properly using `${D}`, so renaming it can break them. Leave `D` at its default.

---

## Assignment Operators

| Operator | Behavior |
|----------|----------|
| `?=` | Default: only assigns if the variable is currently unset |
| `??=` | Weak default, lower priority than `?=`. When both appear, the first `?=` wins |
| `=` | Simple assignment, evaluated at the **end** of parsing (lazy) |
| `:=` | Immediate assignment, evaluated **right now** |
| `+=` | Append with a space |
| `=+` | Prepend with a space |
| `.=` | Append without a space |
| `=.` | Prepend without a space |
| `:append` | Append without a space, applied at expansion time |
| `:prepend` | Prepend without a space, applied at expansion time |
| `:remove` | Remove matching tokens, applied at expansion time |

The key distinction is lazy `=` vs immediate `:=`:

```conf
B = "${A}"   # B captures A's value at the END of parsing
B := "${A}"  # B captures A's current value right now
```

Use `:=` when you need to snapshot a value before something else in the file changes it.
