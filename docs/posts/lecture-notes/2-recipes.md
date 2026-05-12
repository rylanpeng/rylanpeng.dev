---
title: "Yocto Basics Part 2: Writing Recipes (Notes from Tech-A-Byte)"
date: 2026-05-11T22:00:00
categories:
  - Yocto
  - Lecture Notes
---

# Yocto Basics Part 2: Writing Recipes (Notes from Tech-A-Byte)

Part 2 covers writing `.bb` recipes from scratch: the six standard tasks that every recipe goes through, how to declare build and runtime dependencies, how to use Makefiles via `oe_runmake`, and how to extend or mask recipes without touching the originals.

<!-- more -->

---

!!! note "Note"
    These notes follow the [Tech-A-Byte Yocto lecture series](https://youtube.com/playlist?list=PLwqS94HTEwpQmgL1UsSwNk_2tQdzq3eVJ) ([GitHub](https://github.com/Munawar-git/YoctoTutorials/tree/master)) using BeagleBone Yocto as the target board. Replace `<your-path>` with your actual workspace root.

## Creating a Recipe

Recap of the hierarchy: **`poky` → `layer` → `recipe` → `.bb`**

A minimal recipe directory looks like:

```
<your-path>/poky/meta-mylayer/recipes-example/helloworld/
├── files/
│   └── helloworld.c
└── helloworld.bb
```

**Required metadata variables:**

**`SUMMARY`**: a one-line description of the recipe.  
**`LICENSE`**: the license type (MIT, GPL-2.0-only, BSD, etc.).  
**`LIC_FILES_CHKSUM`**: path to the license file plus its md5 checksum. Calculate the checksum with `md5sum`.

```bash
$ cd <your-path>/poky/build
$ bitbake -e helloworld | grep ^COREBASE=
# COREBASE="<your-path>/poky"

$ cd <your-path>/poky/meta
$ md5sum COPYING.MIT
# 3da9cfbcb788c80a0384361b4de20420  COPYING.MIT
```

```conf
LIC_FILES_CHKSUM = "file://${COREBASE}/meta/COPYING.MIT;md5=3da9cfbcb788c80a0384361b4de20420"
```

**`SRC_URI`**: declares source files. The `file://` prefix points to the `files/` subdirectory inside the recipe folder by default.

A minimal helloworld recipe:

```conf
SUMMARY = "Hello World application"
LICENSE = "MIT"
LIC_FILES_CHKSUM = "file://${COREBASE}/meta/COPYING.MIT;md5=3da9cfbcb788c80a0384361b4de20420"

SRC_URI = "file://helloworld.c"

S = "${WORKDIR}/build"

do_compile() {
    ${CC} ${CFLAGS} ${LDFLAGS} ${WORKDIR}/helloworld.c -o ${S}/helloworld
}

do_install() {
    install -d ${D}${bindir}
    install -m 0755 ${S}/helloworld ${D}${bindir}/
}
```

Variable recap:

| Variable | Path |
|----------|------|
| `WORKDIR` | `TMPDIR/<machine>/<PN>/<PV>-<PR>` |
| `S` | `WORKDIR/<PN>-<PV>` (default) |
| `B` | `S` (default) |
| `D` | `WORKDIR/image` |
| `bindir` | `/usr/bin` |

After building, verify:

```bash
$ ls <your-path>/sources/tmp/work/<arch>/helloworld/1.0-r0/build
# helloworld

$ file <your-path>/sources/tmp/work/<arch>/helloworld/1.0-r0/build/helloworld
# ELF 32-bit ARM executable ...

$ ls <your-path>/sources/tmp/work/<arch>/helloworld/1.0-r0/image/usr/bin
# helloworld
```

---

## Recipe Tasks

Every `.bb` goes through a standard sequence of tasks:

| Task | What it does |
|------|-------------|
| `do_fetch` | Download source code into `DL_DIR` |
| `do_unpack` | Unpack source into `S` |
| `do_patch` | Apply patch files to `S` |
| `do_configure` | Configure build-time options |
| `do_compile` | Compile inside `S` |
| `do_install` | Copy results from `S` into `D` |

**Summary:** `S` is where all the active work happens (unpacking, patching, compiling). `D` is the destination: whatever ends up inside `D` is what gets packaged into the final image.

List all tasks for a recipe:

```bash
$ bitbake -c listtasks bbb-example
```

---

### 1. do_fetch

```conf
SRC_URI = "git://github.com/Munawar-git/bbb-tutorial-code.git;protocol=https;branch=master"
SRCREV = "f1be90c683cc1fe31f58f3f8b081562e8b5c8230"
```

```bash
$ bitbake -c do_fetch bbb-example
```

Downloaded source lands in `DL_DIR` (set in `local.conf`):

```bash
$ cd <your-path>/sources/downloads/git2
$ ls | grep bbb
```

---

### 2. do_unpack

```bash
$ bitbake -c do_unpack bbb-example
```

Unpacks the source into `S`.

---

### 3. do_patch

```conf
SRC_URI:append = " file://0001-patch-example.patch"
```

During `do_unpack`, patches listed in `SRC_URI` are placed into `WORKDIR`. `do_patch` then applies them directly to `S`.

```bash
$ bitbake -c do_patch bbb-example
```

---

### 4. do_configure

```conf
do_configure() {
    echo "#define WELCOME y" > bbb-example.h
    echo "#define HELLO y" >> bbb-example.h
}
```

(`>` creates or overwrites, `>>` appends)

```bash
$ bitbake -c do_configure bbb-example
```

---

### 5. do_compile

```conf
do_compile() {
    ${CC} ${CFLAGS} ${LDFLAGS} ${S}/bbb-example.c -o bbb-example
}
```

```bash
$ bitbake -e bbb-example | grep ^CC=
$ bitbake -c do_compile bbb-example
```

`do_compile` runs in `S`, so `-o bbb-example` places the binary in `S`. With `S = "${WORKDIR}/git"`:

```bash
$ cd <your-path>/sources/tmp/work/<arch>/bbb-example/1.0-r0/git
$ file bbb-example
# ELF 32-bit ARM executable ...
```

---

### 6. do_install

```conf
do_install() {
    install -d ${D}${bindir}
    install -m 0755 ${S}/bbb-example ${D}${bindir}
}
```

`bindir` = `/usr/bin`. `do_install` puts everything into `D`. Default `D` = `WORKDIR/image`.

---

## Dependencies

| Variable | Purpose |
|----------|---------|
| `RDEPENDS` | Runtime dependency: A must be installed on the device for B to run |
| `RPROVIDES` | Runtime alias: lets others refer to this package by another name |
| `RCONFLICTS` | Runtime conflict: only one of the conflicting packages can be in `IMAGE_INSTALL` |
| `DEPENDS` | Build dependency: A's libs/headers are needed to compile B |
| `PROVIDES` | Build alias: lets other recipes depend on this one by a virtual name |
| `PREFERRED_PROVIDER_xxx` | If multiple recipes share a `PROVIDES` name, set this in `local.conf` to pick one |
| `PREFERRED_VERSION_xxx` | Pin a specific version, e.g. `PREFERRED_VERSION_add = "1.0.%"` (also in `local.conf`) |

```conf
RDEPENDS:${PN} = "libfoo libbar openssl"
```

```conf
RPROVIDES:${PN} = "bash libfoo"
```

```conf
RCONFLICTS:${PN} = "rconflicts-example2"
RCONFLICTS:${PN} = "rconflicts-example2 rconflicts-example1 (>= 1.2)"
# conflict with example2, and with example1 only if example1 version >= 1.2
```

---

### Q&A: Dependencies under the hood

**Q:** I understand `DEPENDS` vs `RDEPENDS` conceptually, but how does Yocto "not include A" in the final image while still giving B the static library it needs at compile time?

When B declares `DEPENDS = "A"`, BitBake builds A first, then runs A's `do_populate_sysroot` task. This copies A's headers and `.a`/`.so` files from A's `${D}` into a **shared sysroot**, not the final rootfs. B's `do_compile` runs with that sysroot in its search paths, so linking against A's library works. After B is compiled, A's package is never placed into the final image. That's the entire point of `DEPENDS`: "I need A to build, but the device doesn't need A installed at runtime."

`RDEPENDS` is different: it tells the package manager "when you install B on the target device, also install A."

---

**Q:** For `PROVIDES`, does the alias name need to match the actual output library filename?

No. `PROVIDES` is a purely symbolic alias. You can write `PROVIDES = "haha"` and another recipe can write `DEPENDS = "haha"`. BitBake resolves it correctly. Filenames are irrelevant. Convention is to use something meaningful like `virtual/libXXX` for clarity, but it's not enforced.

---

## oe_runmake & EXTRA_OEMAKE

For recipes that use a `Makefile`, use `oe_runmake` instead of calling `make` directly. It automatically injects the correct cross-compilation flags.

```conf
SRC_URI = "file://adv-calculator.c \
           file://adv-math-lib.c \
           file://Makefile \
           "
do_compile() {
    oe_runmake
}
do_install() {
    oe_runmake install DESTDIR=${D} BINDIR=${bindir}
}
```

Use `EXTRA_OEMAKE` to avoid repeating arguments across multiple tasks:

```conf
SRC_URI = "file://adv-calculator.c \
           file://adv-math-lib.c \
           file://Makefile \
           "
EXTRA_OEMAKE:append = " DESTDIR=${D} BINDIR=${bindir}"

do_compile() {
    oe_runmake
}
do_install() {
    oe_runmake install
}
```

---

## BBMASK

To exclude a recipe from the build without removing it from `IMAGE_INSTALL`:

```conf
BBMASK:append = " meta-mylayer/recipes-example/bbb-example"
BBMASK:append = " meta-mylayer/recipes-example/depends-example/subtract.bb"
```

BitBake skips any recipe whose path matches a `BBMASK` pattern.

---

## .bbappend

A `.bbappend` file extends an existing recipe without modifying it. The filename must match the `.bb` it targets. Yocto automatically merges `{name}.bb` and `{name}.bbappend`.

In `bbb-example.bbappend`:

```conf
do_configure:append() {
    echo "#define GOODBYE y" >> bbb-example.h
}
```

---

## FILESEXTRAPATHS

When a `.bbappend` adds files via `SRC_URI`, those files need to be discoverable. By default, BitBake only searches the `files/` directory of the original recipe. If your appended files live in the `.bbappend`'s own directory, you must prepend the search path first.

Say the structure is:

```
meta-mylayer/
├── bbb-example/
│   ├── files/
│   │   └── 0001-original.patch
│   └── bbb-example.bb
└── bbb-example-append/
    ├── bbb-example/          <-- same name as PN
    │   └── 0001-patch-example.patch
    └── bbb-example.bbappend
```

Without `FILESEXTRAPATHS`, the patch won't be found:

```conf
# won't work on its own:
SRC_URI:append = " file://0001-patch-example.patch"
```

Add this to prepend the search path:

```conf
FILESEXTRAPATHS:prepend := "${THISDIR}/${PN}:"

SRC_URI:append = " file://0001-patch-example.patch"
do_configure:append() {
    echo "#define GOODBYE y" >> bbb-example.h
}
```

`${THISDIR}` resolves to the directory containing the `.bbappend` file. `${PN}` is the package name, which is why the subfolder needs to match the package name exactly.

---

## Cleanup

Yocto builds accumulate three main types of artifacts: downloads, sstate cache, and tmp work directories.

| Command | What it cleans |
|---------|---------------|
| `bitbake -c clean` | Per-recipe `WORKDIR` entries in `tmp/` only |
| `bitbake -c cleansstate` | Same as above, plus the recipe's `sstate-cache` entries |
| `bitbake -c cleanall` | Same as cleansstate, plus the recipe's downloaded sources |

```bash
$ bitbake bbb-example

# clean tmp work only
$ bitbake -c clean bbb-example

# check sstate before/after
$ find <your-path>/sources/sstate-cache -iname "*bbb*"
$ bitbake -c cleansstate bbb-example

# check downloads before/after
$ find <your-path>/sources/downloads -iname "*bbb*"
$ bitbake -c cleanall bbb-example
```
