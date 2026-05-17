---
title: "BitBake recipe pipeline: RunQueue and the Tegra flash artifact"
date: 2026-05-17T12:00:00
categories:
  - Jetson Orin Nano
  - Yocto
  - Deep Dive
---

# BitBake recipe pipeline: RunQueue and the Tegra flash artifact

The previous post, [BitBake recipe pipeline: parsing demo-image-weston](./7-bitbake-image-recipe-parse-demo-image-weston.md), ended with the image recipe metadata fully assembled. This post follows the last step. RunQueue expands `demo-image-weston:do_build` into ordered task execution, then the generated Tegra image task creates the flashable `.tegraflash-tar.zst` file.

<!-- more -->

---

!!! note
    BitBake source links point to commit [`941cfe7`](https://git.openembedded.org/bitbake/log/?id=941cfe74c19e01e977101e95b09425d21faf844d) (`yocto-6.0_M3`). openembedded-core source links point to commit [`42fa856a`](https://git.openembedded.org/openembedded-core/log/?id=42fa856a00ac16b2a7a83d7ecfa60a5be192b16c). meta-tegra links point to commit [`2a7fc6f8`](https://github.com/OE4T/meta-tegra/commit/2a7fc6f87d8798355f24eca71c43e49461ac3409).

!!! warning
    This post is my own trace through the source code. I followed the call chain carefully, but my interpretation of what each part does may be incorrect. If you spot an error, please let me know.

---

## Roadmap

```
1.  base.bbclass:6             BB_DEFAULT_TASK ?= "build"
                                → the UI asks for build when no -c task is passed
2.  cooker.py:1528             "build" becomes "do_build"
                                → the runlist starts at demo-image-weston:do_build
3.  cooker.py:1583             RunQueue is created
                                → parsed task metadata becomes a schedulable graph
4.  runqueue.py:750            dependency edges are gathered
                                → addtask, deptask, rdeptask, and recrdeptask are followed
5.  base.bbclass               normal recipe chains
                                → package recipes provide outputs needed by the image
6.  image.bbclass              image recipe chain
                                → rootfs, image, Tegra tar, image complete
7.  image_types_tegra.bbclass  create_tegraflash_pkg
                                → stages flash files and writes the tar
8.  image.bbclass              zst conversion
                                → tegraflash-tar becomes tegraflash-tar.zst
9.  deploy directory           final artifact
                                → unpacking creates the flash_weston directory
```

---

## `do_build` is an aggregate target

When no `-c <task>` option is passed, BitBake uses the default task:

**[`meta/classes-global/base.bbclass:6`](https://git.openembedded.org/openembedded-core/tree/meta/classes-global/base.bbclass?id=42fa856a00ac16b2a7a83d7ecfa60a5be192b16c#n6)**
```python
BB_DEFAULT_TASK ?= "build"
```

The UI reads that value from the server and sends `"build"` in the `buildTargets` command. On the server side, `BBCooker.buildTargets()` normalizes it:

**[`lib/bb/cooker.py:1528`](https://git.openembedded.org/bitbake/tree/lib/bb/cooker.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n1528)**
```python
if task is None:
    task = self.configuration.cmd

if not task.startswith("do_"):
    task = "do_%s" % task
```

So the requested target becomes:

```text
demo-image-weston:do_build
```

That does not mean `do_build` is a large shell function that manually calls every other task. `do_build` is an aggregate task. It is useful because other metadata connects many prerequisite tasks to it.

For normal recipes, `base.bbclass` marks `do_build` as a no op and gives it recursive dependency behavior:

**[`meta/classes-global/base.bbclass:355`](https://git.openembedded.org/openembedded-core/tree/meta/classes-global/base.bbclass?id=42fa856a00ac16b2a7a83d7ecfa60a5be192b16c#n355)**
```python
addtask configure after do_patch
addtask compile after do_configure
addtask install after do_compile
addtask build after do_populate_sysroot
do_build[noexec] = "1"
do_build[recrdeptask] += "do_deploy"
```

The work happens in the tasks that must be complete before `do_build` can be considered done.

---

## RunQueue turns metadata into order

After target resolution, the cooker has a runlist entry for the image:

```python
[
    "",
    "demo-image-weston",
    "do_build",
    ".../layers/meta-tegrademo/recipes-demo/images/demo-image-weston.bb",
]
```

It constructs a `RunQueue` from that runlist:

**[`lib/bb/cooker.py:1583`](https://git.openembedded.org/bitbake/tree/lib/bb/cooker.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n1583)**
```python
rq = bb.runqueue.RunQueue(self, self.data, self.recipecaches, taskdata, runlist)
```

RunQueue walks the parsed task metadata. It does not rely on one hardcoded global order. Instead, each parsed recipe and class has already contributed task nodes and dependency edges.

The dependency gathering code reads those edges:

**[`lib/bb/runqueue.py:750`](https://git.openembedded.org/bitbake/tree/lib/bb/runqueue.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n750)**
```python
task_deps = self.dataCaches[mc].task_deps[taskfn]

for t in taskData[mc].taskentries[tid].tdepends:
    depends.add(build_tid(depmc, depfn, deptaskname))

if 'deptask' in task_deps and taskname in task_deps['deptask']:
    add_build_dependencies(taskData[mc].depids[taskfn], tasknames, depends, mc)

if 'rdeptask' in task_deps and taskname in task_deps['rdeptask']:
    add_runtime_dependencies(taskData[mc].rdepids[taskfn], tasknames, depends, mc)

if 'recrdeptask' in task_deps and taskname in task_deps['recrdeptask']:
    add_build_dependencies(...)
    add_runtime_dependencies(...)
```

The four sources of ordering are:

| Source | What it means |
|--------|---------------|
| `addtask ... after ... before ...` | Ordering within one recipe |
| `[depends]` | Direct task dependency |
| `[deptask]` | Build dependency task edge across recipes |
| `[rdeptask]` and `[recrdeptask]` | Runtime and recursive dependency task edges |

Starting from `demo-image-weston:do_build`, RunQueue follows those edges backward to find everything that must exist first. Then it schedules runnable tasks as their dependencies complete.

---

## Normal recipes supply packages

Most package recipes inherit the global base task chain. It begins with fetch and unpack:

**[`meta/classes-global/base.bbclass:166`](https://git.openembedded.org/openembedded-core/tree/meta/classes-global/base.bbclass?id=42fa856a00ac16b2a7a83d7ecfa60a5be192b16c#n166)**
```python
addtask fetch
```

**[`meta/classes-global/base.bbclass:203`](https://git.openembedded.org/openembedded-core/tree/meta/classes-global/base.bbclass?id=42fa856a00ac16b2a7a83d7ecfa60a5be192b16c#n203)**
```python
addtask unpack after do_fetch
```

Then the common build chain continues:

**[`meta/classes-global/base.bbclass:355`](https://git.openembedded.org/openembedded-core/tree/meta/classes-global/base.bbclass?id=42fa856a00ac16b2a7a83d7ecfa60a5be192b16c#n355)**
```python
addtask configure after do_patch
addtask compile after do_configure
addtask install after do_compile
addtask build after do_populate_sysroot
```

Other classes add the remaining packaging edges. Those package output tasks matter to the image recipe because `do_rootfs` needs installable packages before it can assemble the root filesystem.

So the image target pulls in more than image tasks. It also pulls in package producing recipes needed by `IMAGE_INSTALL`, package manager output such as RPM files, and any build tools that those recipes depend on.

---

## The image recipe runs near the end

The image recipe has its own local task chain from `image.bbclass`:

**[`meta/classes-recipe/image.bbclass:272`](https://git.openembedded.org/openembedded-core/tree/meta/classes-recipe/image.bbclass?id=42fa856a00ac16b2a7a83d7ecfa60a5be192b16c#n272)**
```python
addtask rootfs after do_prepare_recipe_sysroot
```

**[`meta/classes-recipe/image.bbclass:284`](https://git.openembedded.org/openembedded-core/tree/meta/classes-recipe/image.bbclass?id=42fa856a00ac16b2a7a83d7ecfa60a5be192b16c#n284)**
```python
addtask do_image after do_rootfs
```

**[`meta/classes-recipe/image.bbclass:314`](https://git.openembedded.org/openembedded-core/tree/meta/classes-recipe/image.bbclass?id=42fa856a00ac16b2a7a83d7ecfa60a5be192b16c#n314)**
```python
addtask do_image_complete after do_image before do_build
```

The generated Tegra task is inserted between `do_image` and `do_image_complete`:

**[`meta/classes-recipe/image.bbclass:534`](https://git.openembedded.org/openembedded-core/tree/meta/classes-recipe/image.bbclass?id=42fa856a00ac16b2a7a83d7ecfa60a5be192b16c#n534)**
```python
bb.build.addtask(task, 'do_image_complete', after, d)
```

The relevant image chain is:

```text
do_prepare_recipe_sysroot
  -> do_rootfs
  -> do_image_qa
  -> do_image
  -> do_image_tegraflash_tar
  -> do_image_complete
  -> do_build
```

`do_rootfs` installs packages into `${IMAGE_ROOTFS}`. `do_image` runs image preprocessing hooks. `do_image_tegraflash_tar` stages the NVIDIA flash package. `do_image_complete` finalizes image output and lets sstate deploy artifacts to the machine deploy directory.

---

## The Tegra task stages the flash package

The generated task body is:

```sh
create_tegraflash_pkg
```

That shell function comes from `image_types_tegra.bbclass`:

**[`classes-recipe/image_types_tegra.bbclass:340`](https://github.com/OE4T/meta-tegra/blob/2a7fc6f87d8798355f24eca71c43e49461ac3409/classes-recipe/image_types_tegra.bbclass#L340)**

It first creates a staging directory:

```sh
rm -rf "${WORKDIR}/tegraflash"
mkdir -p "${WORKDIR}/tegraflash"
cd "${WORKDIR}/tegraflash"
```

Then it populates the directory with NVIDIA flash files:

```sh
tegraflash_populate_package
```

Those files include firmware blobs, DTBs, BCT files, flash scripts, and other Tegra boot assets from `${STAGING_DATADIR}/tegraflash`.

The function also copies the initrd flasher:

**[`classes-recipe/image_types_tegra.bbclass:351`](https://github.com/OE4T/meta-tegra/blob/2a7fc6f87d8798355f24eca71c43e49461ac3409/classes-recipe/image_types_tegra.bbclass#L351)**
```sh
cp "${IMAGE_TEGRAFLASH_INITRD_FLASHER}" ./initrd-flash.img
```

`IMAGE_TEGRAFLASH_INITRD_FLASHER` resolves to the prebuilt initrd flash image selected by the machine configuration.

Next, it generates the flash config:

**[`classes-recipe/image_types_tegra.bbclass:356`](https://github.com/OE4T/meta-tegra/blob/2a7fc6f87d8798355f24eca71c43e49461ac3409/classes-recipe/image_types_tegra.bbclass#L356)**
```sh
tegraflash_create_flash_config
```

That creates `flash.xml`, which describes the partition layout that the flashing tool will use.

Then the function writes `.env.initrd-flash`, the environment file consumed by the `initrd-flash` wrapper:

**[`classes-recipe/image_types_tegra.bbclass:370`](https://github.com/OE4T/meta-tegra/blob/2a7fc6f87d8798355f24eca71c43e49461ac3409/classes-recipe/image_types_tegra.bbclass#L370)**

The environment includes the boot device, rootfs image name, partition information, and board specific values.

Finally, `tegraflash_finalize_pkg` writes the tar:

**[`classes-recipe/image_types_tegra.bbclass:197`](https://github.com/OE4T/meta-tegra/blob/2a7fc6f87d8798355f24eca71c43e49461ac3409/classes-recipe/image_types_tegra.bbclass#L197)**
```sh
${IMAGE_CMD_TAR} --sparse --numeric-owner \
    -cf ${IMGDEPLOYDIR}/${IMAGE_NAME}.tegraflash-tar .
```

The output at this point is the uncompressed `tegraflash-tar` file in `${IMGDEPLOYDIR}`.

---

## Compression and deploy produce the final file

The requested image type was not just `tegraflash-tar`. It was `tegraflash-tar.zst`. `image.bbclass` split the base type from the compression suffix while generating the task, and the same machinery wires the conversion command after the tar exists.

Conceptually:

```text
create_tegraflash_pkg
  -> ${IMAGE_NAME}.tegraflash-tar
  -> zstd conversion
  -> ${IMAGE_NAME}.tegraflash-tar.zst
```

Then `do_image_complete` and sstate deployment move the final artifact into the machine deploy directory:

```text
build-weston/tmp/deploy/images/jetson-orin-nano-devkit/
  demo-image-weston-jetson-orin-nano-devkit.rootfs.tegraflash-tar.zst
```

That is the file you unpack before flashing:

```bash
tar xf demo-image-weston-jetson-orin-nano-devkit.rootfs.tegraflash-tar.zst
```

The unpacked directory contains the rootfs image, the initrd flash wrapper, NVIDIA flashing scripts, firmware, DTBs, BCTs, and generated configuration.

| File | Origin |
|------|--------|
| `demo-image-weston.ext4` | Rootfs built by `do_rootfs` |
| `initrd-flash` | Wrapper for the initrd flash flow |
| `initrd-flash.img` | Initrd flash image copied by `create_tegraflash_pkg` |
| `tegraflash.py` | NVIDIA host flashing tool from BSP staging |
| `tegraflash_impl_t234.py` | Tegra234 flashing implementation from BSP staging |
| `tegra234-p3768-0000+p3767-0005-nv-super.dtb` | Kernel device tree selected by machine config |
| `bpmp_t234-TE950M-A1_prod.bin` | NVIDIA firmware blob from BSP staging |
| `flash.xml` | Partition layout generated by `tegraflash_create_flash_config` |
| `.env.initrd-flash` | Environment consumed by `initrd-flash` |

Running `sudo ./initrd-flash` uses those staged files to transfer the boot chain and rootfs to the Jetson in recovery mode according to `flash.xml`.

---

## Where we are

| Thing | State |
|------|-------|
| Requested target | `demo-image-weston:do_build` |
| Execution model | RunQueue walks parsed task dependencies |
| Package recipes | Build packages needed by `IMAGE_INSTALL` |
| Image recipe | Builds rootfs, image files, and Tegra flash tar |
| Tegra packaging | `create_tegraflash_pkg` stages flash files |
| Final artifact | `.rootfs.tegraflash-tar.zst` in `tmp/deploy/images` |

The full path is now complete. Config parsing built the global namespace. Recipe parsing built the cache and task graph. Target resolution selected `demo-image-weston:do_build`. RunQueue executed prerequisites first, then the image tasks near the end, and the generated Tegra task produced the flashable archive.
