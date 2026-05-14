---
title: "BitBake config pipeline: the variable store and the layer map"
date: 2026-05-14T18:00:00
categories:
  - Jetson Orin Nano
  - Yocto
  - Deep Dive
---

# BitBake config pipeline: the variable store and the layer map

The previous post, [BitBake startup: the UI handshake and config trigger](./3-bitbake-startup-handshake.md), traced how the first `getVariable` command from the UI triggered `init_configdata()`, which called `initConfigurationData()`, which called `parseBaseConfiguration()`, which finally called `parseConfigurationFiles()`. This post picks up at the first line of that function. It traces what `parseConfigurationFiles()` starts with, how reading `bblayers.conf` gives BitBake its map of the layer topology, how the ten `layer.conf` files fill in the recipe locations and assemble the `BBPATH` search path, and how that assembled `BBPATH` is immediately used to locate `bitbake.conf`.

<!-- more -->

---

!!! note
    BitBake source links point to commit [`941cfe7`](https://git.openembedded.org/bitbake/log/?id=941cfe74c19e01e977101e95b09425d21faf844d) (`yocto-6.0_M3`). openembedded-core source links point to commit [`42fa856a`](https://git.openembedded.org/openembedded-core/log/?id=42fa856a00ac16b2a7a83d7ecfa60a5be192b16c).

!!! info
    Throughout these posts, `build-weston` is the build directory created by running `source setup-env --machine jetson-orin-nano-devkit --distro tegrademo build-weston`.

!!! warning
    This post is my own trace through the source code. I followed the call chain carefully, but my interpretation of what each part does may be incorrect. If you spot an error, please let me know.

---

## Roadmap

```
1.  cookerdata.py:357  parseConfigurationFiles() entry
                        → createCopy(basedata): writable DataSmart, starts as shell-env snapshot
2.  cookerdata.py:365  _findLayerConf() → parse_config_file(bblayers.conf, data)
                        → BBLAYERS (10 layer paths) and initial BBPATH land in the datastore
3.  cookerdata.py:401  for layer in BBLAYERS: parse_config_file(layer/conf/layer.conf, data)
                        → BBPATH prepended per layer, BBFILES extended, BBFILE_COLLECTIONS registered
                        → runs 10 times; BBPATH is fully assembled after the loop
4.  cookerdata.py:485  parse_config_file("conf/bitbake.conf", data)
                        → relative path resolved via BBPATH
5.  parse/__init__.py:132  resolve_file("conf/bitbake.conf", data)
                        → walks BBPATH left to right; only openembedded-core/meta/ has it
```

---

## What `parseConfigurationFiles()` starts with

`parseConfigurationFiles()` is called with three arguments: `self.prefiles`, `self.postfiles`, and `self.basedata`. All three were prepared earlier by `CookerDataBuilder.__init__()`, which ran at the top of `initConfigurationData()`.

| Argument | Type | Value for a plain `bitbake demo-image-weston` run |
|----------|------|--------------------------------------------------|
| `prefiles` | `list[str]` | `[]` (empty unless `--read FILE` was passed on the CLI) |
| `postfiles` | `list[str]` | `[]` (empty unless `--postread FILE` was passed on the CLI) |
| `basedata` | `DataSmart` | a dict-like object seeded with filtered shell environment variables |

`prefiles` and `postfiles` start as empty lists in `CookerConfiguration.__init__()` and stay empty for a plain run:

**[`lib/bb/cookerdata.py:121–126`](https://git.openembedded.org/bitbake/tree/lib/bb/cookerdata.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n121)**
```python
def __init__(self):
    ...
    self.prefile = []
    self.postfile = []
```

`CookerDataBuilder.__init__()` picks them up and builds `basedata` in three sub-steps: create an empty `DataSmart`, copy approved shell variables into it, then tag it with two special keys:

**[`lib/bb/cookerdata.py:225–259`](https://git.openembedded.org/bitbake/tree/lib/bb/cookerdata.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n225)**
```python
def __init__(self, cookercfg, worker=False):
    self.prefiles = cookercfg.prefile
    self.postfiles = cookercfg.postfile
    ...
    self.basedata = bb.data.init()                                      # empty DataSmart
    ...
    bb.data.inheritFromOS(self.basedata, self.savedenv, filtered_keys)  # copy shell vars
    self.basedata.setVar("BB_ORIGENV", self.savedenv)    # full unfiltered env, stored separately
    self.basedata.setVar("__bbclasstype", "global")      # marks this as config context
```

`bb.data.init()` creates the empty store:

**[`lib/bb/data.py:40–42`](https://git.openembedded.org/bitbake/tree/lib/bb/data.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n40)**
```python
def init():
    return _dict_type()    # _dict_type is an alias for DataSmart
```

Here is what `basedata` looks like at the moment `parseConfigurationFiles()` is called, before a single `.conf` file has been read:

```python
# basedata at entry (shown as a plain dict for clarity)
{
    "HOME":          "/home/hello",
    "PATH":          "/usr/bin:/bin:/usr/sbin:/sbin",
    "USER":          "hello",
    "SHELL":         "/bin/bash",
    "TERM":          "xterm-256color",
    # ... ~25 more approved shell vars (PWD, LOGNAME, LANG, etc.) ...
    "BB_ORIGENV":    <full unfiltered shell env, stored as a nested DataSmart>,
    "__bbclasstype": "global",
    # Nothing else. No MACHINE, DISTRO, BBLAYERS, BBPATH, BB_NUMBER_THREADS...
    # All of those will be added by config files below.
}
```

The entire work of this function is expanding this sparse snapshot into a full variable namespace by running every config file through it.

---

## The `DataSmart` variable store

Before following what the config files do, it helps to understand `DataSmart`, because every read and write in the rest of this post and the next goes through it.

`DataSmart` is BitBake's variable store. Think of it as a dict that additionally understands `${VAR}` expansion and BitBake's override operators (`:append`, `:prepend`, `?=`, etc.). Every assignment line in a `.conf` file calls `data.setVar("KEY", "value")` on the active `DataSmart`. Reading a variable calls `data.getVar("KEY")`, which resolves `${VAR}` references at expansion time. When this series says "namespace" or "datastore", it always means a `DataSmart` object.

After all config files have been read, there is one global `DataSmart` stored as `self.data`:

**[`lib/bb/cooker.py:301`](https://git.openembedded.org/bitbake/tree/lib/bb/cooker.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n301)**
```python
self.data = self.databuilder.data
```

When BitBake later parses recipes, each recipe gets its own copy of this global object rather than modifying the original. The copy inherits everything the global store had, and the recipe adds its own variables on top:

**[`lib/bb/cookerdata.py:541–542`](https://git.openembedded.org/bitbake/tree/lib/bb/cookerdata.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n541)**
```python
bb_data = self.data.createCopy()
return self._parse_recipe(bb_data, bbfile, appends, '', layername)
```

The global `self.data` stays frozen for the entire duration of recipe parsing. For example, after parsing `demo-image-weston.bb`, the picture is:

```
self.data  (global, frozen)                bb_data  (per-recipe copy)
──────────────────────────────────         ────────────────────────────────────────
MACHINE  = "jetson-orin-nano-devkit" ───>  MACHINE  = "jetson-orin-nano-devkit"
DISTRO   = "tegrademo"               ───>  DISTRO   = "tegrademo"
... hundreds more config vars ...    ───>  ... same ...
                                           PN = "demo-image-weston"  (added by recipe)
                                           PV = "1.0"                (added by recipe)
```

With `DataSmart` in hand, the entry point of `parseConfigurationFiles()` makes sense immediately.

---

## The writable copy and the config pipeline pattern

The first thing `parseConfigurationFiles()` does is make a writable copy of `basedata`:

**[`lib/bb/cookerdata.py:357–363`](https://git.openembedded.org/bitbake/tree/lib/bb/cookerdata.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n357)**
```python
def parseConfigurationFiles(self, prefiles, postfiles, mc=""):
    data = bb.data.createCopy(self.basedata)
    data.setVar("BB_CURRENT_MC", mc)

    for f in prefiles:
        data = parse_config_file(f, data)
```

Every subsequent config file mutates this copy. `basedata` itself is never touched. For a plain `bitbake` run, `prefiles` is empty, so the loop does nothing.

Every config file in this pipeline goes through the same helper function:

**[`lib/bb/cookerdata.py:179–181`](https://git.openembedded.org/bitbake/tree/lib/bb/cookerdata.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n179)**
```python
def parse_config_file(fn, data, include=True):
    return bb.parse.handle(fn, data, include, baseconfig=True)
```

`baseconfig=True` tells the parser this is a config file, not a recipe. It enables `.conf`-style syntax and suppresses recipe-only constructs. This same call is used for `bblayers.conf`, every `layer.conf`, `bitbake.conf`, and every file `bitbake.conf` includes.

---

## `bblayers.conf`: BitBake's first look at the build

After the empty `prefiles` loop, `parseConfigurationFiles()` looks for `bblayers.conf`. This is the very first real file BitBake reads. The function `_findLayerConf` searches the current working directory (`build-weston/`) for `conf/bblayers.conf`:

**[`lib/bb/cookerdata.py:354–355`](https://git.openembedded.org/bitbake/tree/lib/bb/cookerdata.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n354)**
```python
def _findLayerConf(self, data):
    return findConfigFile("bblayers.conf", data)
```

**[`lib/bb/cookerdata.py:365–371`](https://git.openembedded.org/bitbake/tree/lib/bb/cookerdata.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n365)**
```python
layerconf = self._findLayerConf(data)
if layerconf:
    ...
    data = parse_config_file(layerconf, data)
```

`parse_config_file` reads and executes the entire file. Here is `bblayers.conf` for this build:

**[`build-weston/conf/bblayers.conf:6–20`](https://github.com/rylanpeng/rylanpeng.dev/blob/main/snippets/bblayers.conf#L6)**
```sh
BBPATH = "${TOPDIR}"

BBLAYERS ?= " \
  .../layers/meta \
  .../layers/meta-tegra \
  .../layers/meta-oe \
  .../layers/meta-python \
  .../layers/meta-networking \
  .../layers/meta-filesystems \
  .../layers/meta-tegra-community \
  .../layers/meta-tegra-support \
  .../layers/meta-demo-ci \
  .../layers/meta-tegrademo \
  "
```

After this file is parsed, two variables are in the datastore. `BBLAYERS` is a space-separated list of ten absolute paths, one per layer. `BBPATH` is set to `"${TOPDIR}"`, which will expand to the `build-weston/` root.

`bblayers.conf` is read first because `BBLAYERS` is what the next step needs. Without it, BitBake does not know where any layer lives and cannot proceed. The `BBPATH` initialization also matters here: the next step prepends each layer directory to `BBPATH`, and by the end of that loop, `BBPATH` will be the key used to locate `bitbake.conf`.

---

## The `layer.conf` loop: BBPATH, BBFILES, and layer priorities

With `BBLAYERS` in the datastore, `parseConfigurationFiles()` iterates over every layer path and reads the `layer.conf` inside it:

**[`lib/bb/cookerdata.py:401–409`](https://git.openembedded.org/bitbake/tree/lib/bb/cookerdata.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n401)**
```python
for layer in layers:    # layers = paths from BBLAYERS
    ...
    data = parse_config_file(os.path.join(layer, "conf", "layer.conf"), data)
```

This loop runs ten times, once per layer. `parse_config_file()` reads and executes the entire `layer.conf` on each iteration. Here is `meta-tegrademo`'s `layer.conf` as a representative example:

**[`layers/meta-tegrademo/conf/layer.conf`](https://github.com/OE4T/tegra-demo-distro/blob/d48b1464061b56b7f679731dbd8b7747c048d1b7/layers/meta-tegrademo/conf/layer.conf)**
```sh
BBPATH =. "${LAYERDIR}:"
BBFILES += "${LAYERDIR}/recipes-*/*/*.bb ${LAYERDIR}/recipes-*/*/*.bbappend"
BBFILE_COLLECTIONS += "tegrademo"
BBFILE_PRIORITY_tegrademo = "50"
```

Every `layer.conf` in this build is structurally identical. The only differences are the short name registered in `BBFILE_COLLECTIONS` and the priority number. Each line serves a distinct purpose.

**`BBPATH =. "${LAYERDIR}:"`**

The `=.` operator is a prepend: it places `${LAYERDIR}` at the front of `BBPATH`. `LAYERDIR` is automatically set to the absolute path of the layer being processed. Since the loop runs through all ten layers in `BBLAYERS` order, each one prepends its own path to `BBPATH`. After all ten iterations, `BBPATH` looks like this:

```
meta-tegrademo: ... :meta-tegra:meta:build-weston
```

`build-weston` is at the far right because it was set first by `bblayers.conf` and was never prepended over. The rightmost entry is searched last. This ordering matters when BitBake looks up a relative file path: it walks `BBPATH` left to right and stops at the first match.

**`BBFILES += "${LAYERDIR}/recipes-*/*/*.bb ${LAYERDIR}/recipes-*/*/*.bbappend"`**

`BBFILES` accumulates glob patterns as each `layer.conf` runs. By the time the loop finishes, `BBFILES` is a space-separated string of glob patterns covering every layer. When BitBake parses recipes, it expands all these globs to build the complete list of recipe files to read.

**`BBFILE_COLLECTIONS += "tegrademo"` and `BBFILE_PRIORITY_tegrademo = "50"`**

`BBFILE_COLLECTIONS` is a name registry. Each layer adds its own short name. The name itself does nothing directly: its only purpose is to act as the key for a corresponding `BBFILE_PRIORITY_<name>` variable. Without the name in `BBFILE_COLLECTIONS`, BitBake would not know that `BBFILE_PRIORITY_tegrademo` refers to the `meta-tegrademo` layer.

When two layers both provide a recipe file with the same name, BitBake picks the one from the layer with the higher priority number. If `meta` has priority 5 and `meta-tegra` has priority 10 and both contain a `linux-yocto_%.bbappend`, BitBake uses `meta-tegra`'s version and silently ignores `meta`'s. Every layer registers exactly one name because priority is a per-layer concept.

---

## After the loop: BBPATH is ready

After all ten `layer.conf` files are read, the datastore holds the full layer topology:

```python
{
    # ... shell environment vars from basedata ...
    "BBPATH":              "meta-tegrademo: ... :meta-tegra:meta:build-weston",
    "BBLAYERS":            "/path/to/meta /path/to/meta-tegra ... /path/to/meta-tegrademo",
    "BBFILES":             "${meta}/recipes-*/*/*.bb ... ${meta-tegrademo}/recipes-*/*/*.bbappend",
    "BBFILE_COLLECTIONS":  "core base ... tegra tegrademo",
    "BBFILE_PRIORITY_core":     "5",
    "BBFILE_PRIORITY_tegra":    "10",
    "BBFILE_PRIORITY_tegrademo":"50",
    # ... one BBFILE_PRIORITY_<name> per layer ...
}
```

`MACHINE`, `DISTRO`, `IMAGE_FSTYPES`, and every other build-configuration variable are still absent. Those come from `bitbake.conf` and the files it includes. The first step toward getting there is finding `bitbake.conf`, and `BBPATH` is exactly what makes that possible.

---

## Finding `bitbake.conf` via BBPATH

After the `layer.conf` loop, `parseConfigurationFiles()` calls:

**[`lib/bb/cookerdata.py:485`](https://git.openembedded.org/bitbake/tree/lib/bb/cookerdata.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n485)**
```python
data = parse_config_file(os.path.join("conf", "bitbake.conf"), data)
```

`"conf/bitbake.conf"` is a relative path. BitBake does not hard-code where this file lives. Why does it resolve to `openembedded-core/meta/conf/bitbake.conf` specifically? The answer follows directly from how `BBPATH` was assembled in the steps above.

1. [`bblayers.conf:6`](https://github.com/rylanpeng/rylanpeng.dev/blob/main/snippets/bblayers.conf#L6) starts with `BBPATH = "${TOPDIR}"`. After that line, `BBPATH` expands to `"/path/to/build-weston"`.
2. The `layer.conf` loop then parses each layer in `BBLAYERS` order (`meta`, `meta-tegra`, `meta-oe`, ...) and each one does `BBPATH =. "${LAYERDIR}:"`. The `=.` is a prepend, so every new layer lands at the front. After all ten layers have run, `BBPATH` is `"meta-tegrademo: … :meta-tegra:meta:build-weston"`.
3. `parse_config_file` calls `bb.parse.handle`, which calls `resolve_file`. Inside `resolve_file`, `bb.utils.which(BBPATH, "conf/bitbake.conf")` walks `BBPATH` entries left to right and stops at the first entry whose directory contains `conf/bitbake.conf`.
4. Only `openembedded-core/meta/conf/bitbake.conf` exists. None of the other layers ship a `conf/bitbake.conf`. That is the oe-core convention: every Yocto project has exactly one `bitbake.conf`, always in the `meta` layer of openembedded-core. BitBake does not special-case the path; it just finds it first because it is the only one.

Here is `resolve_file` implementing step 3:

**[`lib/bb/parse/__init__.py:132–147`](https://git.openembedded.org/bitbake/tree/lib/bb/parse/__init__.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n132)**
```python
def resolve_file(fn, d):
    if not os.path.isabs(fn):
        bbpath = d.getVar("BBPATH")
        newfn, attempts = bb.utils.which(bbpath, fn, history=True)
        ...
        fn = newfn
```

This same `BBPATH`-plus-filename search is the general pattern for every relative conf file lookup in this pipeline. `findConfigFile` uses the identical walk:

**[`lib/bb/cookerdata.py:188–193`](https://git.openembedded.org/bitbake/tree/lib/bb/cookerdata.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n188)**
```python
def findConfigFile(configfile, data):
    search = []
    bbpath = data.getVar("BBPATH")
    if bbpath:
        for i in bbpath.split(":"):
            search.append(os.path.join(i, "conf", configfile))
```

So `BBPATH` does double duty: the `layer.conf` loop builds it as a side effect of registering each layer, and then the very next step relies on it to resolve the path to `bitbake.conf`. The loop and the lookup form a single continuous chain.

`bitbake.conf` is now found and parsed. It is roughly 900 lines of build-system defaults, but what matters for this trace is its tail: a chain of `include` and `require` directives that pull in `local.conf`, the machine conf, and the distro conf. Those files are where `MACHINE`, `DISTRO`, `IMAGE_FSTYPES`, `KERNEL_DEVICETREE`, and the rest of the project-specific variables are set. That include chain is traced in the next post.
