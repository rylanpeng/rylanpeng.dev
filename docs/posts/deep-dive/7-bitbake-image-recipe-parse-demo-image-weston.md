---
title: "BitBake recipe pipeline: parsing demo-image-weston"
date: 2026-05-17T11:00:00
categories:
  - Jetson Orin Nano
  - Yocto
  - Deep Dive
---

# BitBake recipe pipeline: parsing demo-image-weston

The previous post, [BitBake recipe pipeline: cache build and target resolution](./6-bitbake-recipe-cache-and-target-resolution.md), traced how `demo-image-weston.bb` becomes a provider for the target string `demo-image-weston`. This post zooms into that one recipe parse. It follows the `require` and `inherit` chain that turns a small image recipe into a full task graph with rootfs, image, and Tegra flash packaging tasks.

<!-- more -->

---

!!! note
    BitBake source links point to commit [`941cfe7`](https://git.openembedded.org/bitbake/log/?id=941cfe74c19e01e977101e95b09425d21faf844d) (`yocto-6.0_M3`). openembedded-core source links point to commit [`42fa856a`](https://git.openembedded.org/openembedded-core/log/?id=42fa856a00ac16b2a7a83d7ecfa60a5be192b16c). meta-tegra links point to commit [`2a7fc6f8`](https://github.com/OE4T/meta-tegra/commit/2a7fc6f87d8798355f24eca71c43e49461ac3409). tegra-demo-distro links point to commit [`d48b1464`](https://github.com/OE4T/tegra-demo-distro/commit/d48b1464061b56b7f679731dbd8b7747c048d1b7).

!!! warning
    This post is my own trace through the source code. I followed the call chain carefully, but my interpretation of what each part does may be incorrect. If you spot an error, please let me know.

---

## Roadmap

```
1.  demo-image-weston.bb:3       require demo-image-common.inc
                                  → common image settings are inlined
2.  demo-image-weston.bb:5       IMAGE_FEATURES += "hwcodecs weston"
                                  → image features are requested, not direct package names
3.  demo-image-weston.bb:7       inherit features_check
                                  → REQUIRED_DISTRO_FEATURES is checked at parse time
4.  demo-image-common.inc:5      inherit core-image
                                  → feature names map to package groups
5.  core-image.bbclass:95        inherit image
                                  → image build tasks are defined
6.  image.bbclass:25             inherit_defer ${IMGCLASSES}
                                  → rootfs, image type, Tegra, WIC, and postcommand classes load later
7.  image.bbclass:519            dynamic image task generation
                                  → do_image_tegraflash_tar is created from IMAGE_FSTYPES
8.  rootfs-postcommands.bbclass  rootfs post commands
                                  → systemd default target is written into the rootfs
9.  image_types_tegra.bbclass    IMAGE_CMD:tegraflash-tar
                                  → the generated task calls create_tegraflash_pkg
```

---

## Recipe parsing is not task execution

The recipe parser runs while BitBake is building the recipe cache. It evaluates metadata and records tasks, but it does not run those tasks yet.

So during this parse:

1. No root filesystem is assembled.
2. No packages are installed.
3. No `.tegraflash-tar.zst` file is written.
4. No Jetson flash directory is created.

The output of parsing is a datastore for this recipe. That datastore contains variables, shell functions, Python functions, task definitions, and dependency edges. RunQueue will use that metadata later.

The recipe itself is small:

**[`layers/meta-tegrademo/recipes-demo/images/demo-image-weston.bb`](https://github.com/OE4T/tegra-demo-distro/blob/d48b1464061b56b7f679731dbd8b7747c048d1b7/layers/meta-tegrademo/recipes-demo/images/demo-image-weston.bb)**

It becomes powerful because it pulls in shared image machinery.

---

## The image recipe starts with common metadata

The first line that matters is a `require`:

**[`demo-image-weston.bb:3`](https://github.com/OE4T/tegra-demo-distro/blob/d48b1464061b56b7f679731dbd8b7747c048d1b7/layers/meta-tegrademo/recipes-demo/images/demo-image-weston.bb#L3)**
```python
require demo-image-common.inc
```

`require` inlines the target file at that point in the parse. It is not a second recipe. It does not create a second set of tasks. It is more like `#include` in C.

The included file adds SSH and the common demo package groups:

**[`demo-image-common.inc:1`](https://github.com/OE4T/tegra-demo-distro/blob/d48b1464061b56b7f679731dbd8b7747c048d1b7/layers/meta-tegrademo/recipes-demo/images/demo-image-common.inc#L1)**
```python
IMAGE_FEATURES += "ssh-server-openssh"
```

**[`demo-image-common.inc:7`](https://github.com/OE4T/tegra-demo-distro/blob/d48b1464061b56b7f679731dbd8b7747c048d1b7/layers/meta-tegrademo/recipes-demo/images/demo-image-common.inc#L7)**
```python
CORE_IMAGE_BASE_INSTALL += "packagegroup-demo-base packagegroup-demo-basetests"
CORE_IMAGE_BASE_INSTALL += "${@'packagegroup-demo-systemd' if d.getVar('VIRTUAL-RUNTIME_init_manager') == 'systemd' else ''}"
```

The second line evaluates to `packagegroup-demo-systemd` because the completed config namespace set `INIT_MANAGER = "systemd"`, which makes `VIRTUAL-RUNTIME_init_manager` resolve to `systemd`.

The file also inherits the main OpenEmbedded image class:

**[`demo-image-common.inc:5`](https://github.com/OE4T/tegra-demo-distro/blob/d48b1464061b56b7f679731dbd8b7747c048d1b7/layers/meta-tegrademo/recipes-demo/images/demo-image-common.inc#L5)**
```python
inherit core-image
```

That is the next step because the recipe has now named image features, but it still needs a class that knows how to turn those features into installable package groups.

---

## Image features become package groups

Back in `demo-image-weston.bb`, the recipe adds the two features that make this image different from a minimal image:

**[`demo-image-weston.bb:5`](https://github.com/OE4T/tegra-demo-distro/blob/d48b1464061b56b7f679731dbd8b7747c048d1b7/layers/meta-tegrademo/recipes-demo/images/demo-image-weston.bb#L5)**
```python
IMAGE_FEATURES += "hwcodecs weston"
```

`IMAGE_FEATURES` is a feature list. It is not a package list. The mapping lives in `core-image.bbclass`:

**[`meta/classes-recipe/core-image.bbclass:57`](https://git.openembedded.org/openembedded-core/tree/meta/classes-recipe/core-image.bbclass?id=42fa856a00ac16b2a7a83d7ecfa60a5be192b16c#n57)**
```python
FEATURE_PACKAGES_hwcodecs           = "${MACHINE_HWCODECS}"
FEATURE_PACKAGES_weston             = "packagegroup-core-weston"
FEATURE_PACKAGES_ssh-server-openssh = "packagegroup-core-ssh-openssh"
```

At this point, the important feature set is:

```python
IMAGE_FEATURES = "ssh-server-openssh hwcodecs weston"
```

`core-image.bbclass` also supplies the base install list and connects it to `IMAGE_INSTALL`:

**[`meta/classes-recipe/core-image.bbclass:84`](https://git.openembedded.org/openembedded-core/tree/meta/classes-recipe/core-image.bbclass?id=42fa856a00ac16b2a7a83d7ecfa60a5be192b16c#n84)**
```python
CORE_IMAGE_BASE_INSTALL = '\
    packagegroup-core-boot \
    packagegroup-base-extended \
    ${CORE_IMAGE_EXTRA_INSTALL} \
    '
IMAGE_INSTALL ?= "${CORE_IMAGE_BASE_INSTALL}"
```

This is how the image recipe moves from named features to packages that can be installed into a root filesystem.

---

## The feature check runs during parsing

`demo-image-weston.bb` requires Wayland and OpenGL support:

**[`demo-image-weston.bb:7`](https://github.com/OE4T/tegra-demo-distro/blob/d48b1464061b56b7f679731dbd8b7747c048d1b7/layers/meta-tegrademo/recipes-demo/images/demo-image-weston.bb#L7)**
```python
inherit features_check
```

**[`demo-image-weston.bb:9`](https://github.com/OE4T/tegra-demo-distro/blob/d48b1464061b56b7f679731dbd8b7747c048d1b7/layers/meta-tegrademo/recipes-demo/images/demo-image-weston.bb#L9)**
```python
REQUIRED_DISTRO_FEATURES = "wayland opengl"
```

`features_check.bbclass` reads `REQUIRED_DISTRO_FEATURES` and compares it with `DISTRO_FEATURES` in the recipe datastore:

**[`meta/classes-recipe/features_check.bbclass:41`](https://git.openembedded.org/openembedded-core/tree/meta/classes-recipe/features_check.bbclass?id=42fa856a00ac16b2a7a83d7ecfa60a5be192b16c#n41)**
```python
required_features = set((d.getVar('REQUIRED_' + kind + '_FEATURES') or '').split())
if required_features:
    missing = set.difference(required_features, features)
    if missing:
        raise bb.parse.SkipRecipe(
            "using %s '%s', which is missing required %s_FEATURES: '%s'"
            % (kind, d.getVar(kind), kind, ' '.join(missing)))
```

For this build, `DISTRO_FEATURES` includes `pam systemd usrmerge wayland opengl`. The `wayland opengl` part came from `local.conf`. Both required features are present, so `SkipRecipe` is not raised.

This matters because the failure would happen while parsing metadata, before any package compile starts. A missing `DISTRO_FEATURES:append = " wayland opengl"` line is not a late image failure. BitBake can reject the recipe during cache construction.

---

## `image.bbclass` defines the image task skeleton

`core-image.bbclass` finishes by inheriting `image.bbclass`:

**[`meta/classes-recipe/core-image.bbclass:95`](https://git.openembedded.org/openembedded-core/tree/meta/classes-recipe/core-image.bbclass?id=42fa856a00ac16b2a7a83d7ecfa60a5be192b16c#n95)**
```python
inherit image
```

This is where image construction tasks enter the recipe metadata.

First, `image.bbclass` assembles `IMGCLASSES`:

**[`meta/classes-recipe/image.bbclass:7`](https://git.openembedded.org/openembedded-core/tree/meta/classes-recipe/image.bbclass?id=42fa856a00ac16b2a7a83d7ecfa60a5be192b16c#n7)**
```python
IMAGE_CLASSES ??= ""
IMGCLASSES = "rootfs_${IMAGE_PKGTYPE} image_types ${IMAGE_CLASSES}"
IMGCLASSES += "${@['', 'populate_sdk_ext']['linux' in d.getVar("SDK_OS")]}"
IMGCLASSES += "${@bb.utils.contains_any('IMAGE_FSTYPES', 'live iso hddimg', 'image-live', '', d)}"
IMGCLASSES += "${@bb.utils.contains('IMAGE_FSTYPES', 'container', 'image-container', '', d)}"
IMGCLASSES += "image_types_wic"
IMGCLASSES += "rootfs-postcommands"
```

Then it defers inheritance of those classes:

**[`meta/classes-recipe/image.bbclass:25`](https://git.openembedded.org/openembedded-core/tree/meta/classes-recipe/image.bbclass?id=42fa856a00ac16b2a7a83d7ecfa60a5be192b16c#n25)**
```python
inherit_defer ${IMGCLASSES}
```

`inherit_defer` delays loading until finalization. That gives the expressions above time to finish assembling `IMGCLASSES`.

For this build:

| Variable | Value that matters |
|----------|--------------------|
| `IMAGE_PKGTYPE` | `rpm` |
| `IMAGE_CLASSES` | includes `image_types_tegra` |
| `IMAGE_FSTYPES` | includes `tegraflash-tar.zst` |

So the image recipe pulls in:

```text
rootfs_rpm image_types image_types_tegra image_types_wic rootfs-postcommands
```

The `image_types_tegra` class appears here because the machine configuration added it to `IMAGE_CLASSES` earlier.

---

## The image task chain is recorded

`image.bbclass` defines the core image tasks. The first one builds the root filesystem:

**[`meta/classes-recipe/image.bbclass:205`](https://git.openembedded.org/openembedded-core/tree/meta/classes-recipe/image.bbclass?id=42fa856a00ac16b2a7a83d7ecfa60a5be192b16c#n205)**
```python
fakeroot python do_rootfs () {
    from oe.rootfs import create_rootfs
    from oe.manifest import create_manifest
    ...
    create_rootfs(d, progress_reporter=progress_reporter, logcatcher=logcatcher)
}
```

It is registered after the recipe sysroot is prepared:

**[`meta/classes-recipe/image.bbclass:272`](https://git.openembedded.org/openembedded-core/tree/meta/classes-recipe/image.bbclass?id=42fa856a00ac16b2a7a83d7ecfa60a5be192b16c#n272)**
```python
addtask rootfs after do_prepare_recipe_sysroot
```

The rootfs is checked before image files are created:

**[`meta/classes-recipe/image.bbclass:334`](https://git.openembedded.org/openembedded-core/tree/meta/classes-recipe/image.bbclass?id=42fa856a00ac16b2a7a83d7ecfa60a5be192b16c#n334)**
```python
fakeroot python do_image_qa () {
    qa_cmds = (d.getVar('IMAGE_QA_COMMANDS') or '').split()
    for cmd in qa_cmds:
        bb.build.exec_func(cmd, d)
    oe.qa.exit_if_errors(d)
}
```

**[`meta/classes-recipe/image.bbclass:342`](https://git.openembedded.org/openembedded-core/tree/meta/classes-recipe/image.bbclass?id=42fa856a00ac16b2a7a83d7ecfa60a5be192b16c#n342)**
```python
addtask do_image_qa after do_rootfs before do_image
```

Then `do_image` runs preprocessing hooks:

**[`meta/classes-recipe/image.bbclass:274`](https://git.openembedded.org/openembedded-core/tree/meta/classes-recipe/image.bbclass?id=42fa856a00ac16b2a7a83d7ecfa60a5be192b16c#n274)**
```python
fakeroot python do_image () {
    from oe.utils import execute_pre_post_process
    pre_process_cmds = d.getVar("IMAGE_PREPROCESS_COMMAND")
    execute_pre_post_process(d, pre_process_cmds)
}
```

**[`meta/classes-recipe/image.bbclass:284`](https://git.openembedded.org/openembedded-core/tree/meta/classes-recipe/image.bbclass?id=42fa856a00ac16b2a7a83d7ecfa60a5be192b16c#n284)**
```python
addtask do_image after do_rootfs
```

Finally, `do_image_complete` runs postprocess hooks and sits before `do_build`:

**[`meta/classes-recipe/image.bbclass:286`](https://git.openembedded.org/openembedded-core/tree/meta/classes-recipe/image.bbclass?id=42fa856a00ac16b2a7a83d7ecfa60a5be192b16c#n286)**
```python
fakeroot python do_image_complete () {
    from oe.utils import execute_pre_post_process
    post_process_cmds = d.getVar("IMAGE_POSTPROCESS_COMMAND")
    execute_pre_post_process(d, post_process_cmds)
    ...
}
```

**[`meta/classes-recipe/image.bbclass:314`](https://git.openembedded.org/openembedded-core/tree/meta/classes-recipe/image.bbclass?id=42fa856a00ac16b2a7a83d7ecfa60a5be192b16c#n314)**
```python
addtask do_image_complete after do_image before do_build
```

At this point there is a rootfs and image skeleton. The missing piece is the Tegra flash tar task.

---

## The Tegra flash task is generated

There is no static function named `do_image_tegraflash_tar` in the source tree. It is generated by an anonymous Python block in `image.bbclass`.

The block starts from `IMAGE_FSTYPES`:

**[`meta/classes-recipe/image.bbclass:394`](https://git.openembedded.org/openembedded-core/tree/meta/classes-recipe/image.bbclass?id=42fa856a00ac16b2a7a83d7ecfa60a5be192b16c#n394)**
```python
alltypes = d.getVar('IMAGE_FSTYPES').split()
```

For `tegraflash-tar.zst`, BitBake strips the compression suffix and gets the base type `tegraflash-tar`:

**[`meta/classes-recipe/image.bbclass:380`](https://git.openembedded.org/openembedded-core/tree/meta/classes-recipe/image.bbclass?id=42fa856a00ac16b2a7a83d7ecfa60a5be192b16c#n380)**
```python
def _image_base_type(type):
    for ctype in ctypes:
        if type.endswith("." + ctype):
            basetype = type[:-len("." + ctype)]
            break
    return basetype
```

Then it turns that base type into a task name:

**[`meta/classes-recipe/image.bbclass:519`](https://git.openembedded.org/openembedded-core/tree/meta/classes-recipe/image.bbclass?id=42fa856a00ac16b2a7a83d7ecfa60a5be192b16c#n519)**
```python
task = "do_image_%s" % t.replace("-", "_").replace(".", "_")
```

For `tegraflash-tar`, the result is:

```text
do_image_tegraflash_tar
```

The task body comes from `IMAGE_CMD` under the image type override:

**[`meta/classes-recipe/image.bbclass:467`](https://git.openembedded.org/openembedded-core/tree/meta/classes-recipe/image.bbclass?id=42fa856a00ac16b2a7a83d7ecfa60a5be192b16c#n467)**
```python
image_cmd = localdata.getVar("IMAGE_CMD")
cmds.append("\t" + image_cmd)
d.setVar(task, '\n'.join(cmds))
```

`image_types_tegra.bbclass` supplies that command:

**[`classes-recipe/image_types_tegra.bbclass:458`](https://github.com/OE4T/meta-tegra/blob/2a7fc6f87d8798355f24eca71c43e49461ac3409/classes-recipe/image_types_tegra.bbclass#L458)**
```python
IMAGE_CMD:tegraflash-tar = "create_tegraflash_pkg"
```

Then the generated task is registered:

**[`meta/classes-recipe/image.bbclass:534`](https://git.openembedded.org/openembedded-core/tree/meta/classes-recipe/image.bbclass?id=42fa856a00ac16b2a7a83d7ecfa60a5be192b16c#n534)**
```python
bb.build.addtask(task, 'do_image_complete', after, d)
```

The conceptual result is:

```text
do_image_tegraflash_tar:
    create_tegraflash_pkg
```

It runs after `do_image` and before `do_image_complete`.

---

## The rootfs post command sets the boot target

`rootfs-postcommands.bbclass` contributes commands that run inside `do_rootfs` after packages have been installed. For this image, the key behavior is the systemd default target.

The class sets a default target and appends systemd commands when `DISTRO_FEATURES` contains `systemd`:

**[`meta/classes-recipe/rootfs-postcommands.bbclass:45`](https://git.openembedded.org/openembedded-core/tree/meta/classes-recipe/rootfs-postcommands.bbclass?id=42fa856a00ac16b2a7a83d7ecfa60a5be192b16c#n45)**
```python
SYSTEMD_DEFAULT_TARGET ?= '${@bb.utils.contains_any("IMAGE_FEATURES", ["x11-base", "weston"], "graphical.target", "multi-user.target", d)}'
ROOTFS_POSTPROCESS_COMMAND += '${@bb.utils.contains("DISTRO_FEATURES", "systemd", "set_systemd_default_target systemd_sysusers_check systemd_handle_machine_id", "", d)}'
```

The recipe sets the target directly:

**[`demo-image-weston.bb:14`](https://github.com/OE4T/tegra-demo-distro/blob/d48b1464061b56b7f679731dbd8b7747c048d1b7/layers/meta-tegrademo/recipes-demo/images/demo-image-weston.bb#L14)**
```python
SYSTEMD_DEFAULT_TARGET = "graphical.target"
```

The shell function creates the symlink inside the image rootfs:

**[`meta/classes-recipe/rootfs-postcommands.bbclass:366`](https://git.openembedded.org/openembedded-core/tree/meta/classes-recipe/rootfs-postcommands.bbclass?id=42fa856a00ac16b2a7a83d7ecfa60a5be192b16c#n366)**
```sh
set_systemd_default_target () {
    if [ -d ${IMAGE_ROOTFS}${sysconfdir}/systemd/system -a \
         -e ${IMAGE_ROOTFS}${systemd_system_unitdir}/${SYSTEMD_DEFAULT_TARGET} ]; then
        ln -sf ${systemd_system_unitdir}/${SYSTEMD_DEFAULT_TARGET} \
               ${IMAGE_ROOTFS}${sysconfdir}/systemd/system/default.target
    fi
}
```

Concretely, it creates:

```text
/etc/systemd/system/default.target -> /lib/systemd/system/graphical.target
```

That symlink is why this image boots into the graphical target rather than a plain multi user target.

---

## Where we are

The parse of `demo-image-weston.bb` has created this local task chain:

```text
do_prepare_recipe_sysroot
  -> do_rootfs
  -> do_image_qa
  -> do_image
  -> do_image_tegraflash_tar
  -> do_image_complete
  -> do_build
```

The key result is metadata, not artifacts. `do_image_tegraflash_tar` now exists as a generated task whose body calls `create_tegraflash_pkg`. `do_rootfs` knows how to install the image package set. The rootfs post commands know how to set `graphical.target`. The next post follows how RunQueue turns this parsed task graph into execution and how the flashable tarball appears in `tmp/deploy/images`.
