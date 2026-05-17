---
title: "BitBake recipe pipeline: cache build and target resolution"
date: 2026-05-17T10:00:00
categories:
  - Jetson Orin Nano
  - Yocto
  - Deep Dive
---

# BitBake recipe pipeline: cache build and target resolution

The previous post, [BitBake config pipeline: the bitbake.conf include chain and the complete namespace](./5-bitbake-config-pipeline-bitbake-conf.md), ended with a complete global datastore. This post traces the first half of recipe processing. BitBake discovers every recipe file, parses them into a cache, and only then resolves the command line target `demo-image-weston`.

<!-- more -->

---

!!! note
    BitBake source links point to commit [`941cfe7`](https://git.openembedded.org/bitbake/log/?id=941cfe74c19e01e977101e95b09425d21faf844d) (`yocto-6.0_M3`). openembedded-core source links point to commit [`42fa856a`](https://git.openembedded.org/openembedded-core/log/?id=42fa856a00ac16b2a7a83d7ecfa60a5be192b16c). tegra-demo-distro links point to commit [`d48b1464`](https://github.com/OE4T/tegra-demo-distro/commit/d48b1464061b56b7f679731dbd8b7747c048d1b7).

!!! warning
    This post is my own trace through the source code. I followed the call chain carefully, but my interpretation of what each part does may be incorrect. If you spot an error, please let me know.

---

## Roadmap

```
1.  command.py:632     buildTargets.needcache = True
                        → the requested target cannot run until the recipe cache exists
2.  command.py:110     runAsyncCommand()
                        → calls updateCache() while the cooker is not RUNNING
3.  cooker.py:1631     updateCache()
                        → prepares parsing and collects every .bb file from BBFILES
4.  cooker.py:1842     collect_bbfiles()
                        → expands layer contributed BBFILES globs
5.  cooker.py:1675     CookerParser
                        → parses every recipe, including demo-image-weston.bb
6.  cache.py:174       provider indexes
                        → records demo-image-weston as a provider key
7.  cooker.py:1516     BBCooker.buildTargets()
                        → converts "build" to "do_build"
8.  taskdata.py:337    TaskData.add_provider()
                        → resolves demo-image-weston through the completed cache
```

---

## The command waits behind the cache

The UI has already sent this command to the server:

```python
["buildTargets", ["demo-image-weston"], "build"]
```

It would be easy to assume that BitBake now opens `demo-image-weston.bb` by name. That is not what happens. The command is registered as an asynchronous command, then the command runner checks whether it needs the recipe cache first.

`buildTargets` explicitly says that it does:

**[`lib/bb/command.py:632`](https://git.openembedded.org/bitbake/tree/lib/bb/command.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n632)**
```python
def buildTargets(self, command, params):
    pkgs_to_build = params[0]
    task = params[1]
    command.cooker.buildTargets(pkgs_to_build, task)
buildTargets.needcache = True
```

That flag creates the next question. If the user asked for a build, but BitBake does not yet know which recipes exist, what can the server do first? It can build the cache.

The asynchronous runner checks `needcache`. If the cooker is not in `State.RUNNING`, it calls `updateCache()` and returns. The real `buildTargets()` method is delayed until a later idle pass.

**[`lib/bb/command.py:110`](https://git.openembedded.org/bitbake/tree/lib/bb/command.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n110)**
```python
def runAsyncCommand(self, _, process_server, halt):
    ...
    needcache = getattr(commandmethod, "needcache")
    if needcache and self.cooker.state != bb.cooker.State.RUNNING:
        self.cooker.updateCache()
        return True
    else:
        commandmethod(self.cmds_async, self, options)
        return False
```

So `buildTargets` is the next command in the control flow, but it is not doing target resolution yet. It is the reason cache construction starts.

---

## Recipe discovery starts from `BBFILES`

`updateCache()` is the coordinator. It refreshes the cache if needed, applies multiconfig settings, then asks the layer collection to collect recipe files.

**[`lib/bb/cooker.py:1631`](https://git.openembedded.org/bitbake/tree/lib/bb/cooker.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n1631)**
```python
def updateCache(self):
    if self.state == State.RUNNING:
        return
    ...
    if self.state != State.PARSING:
        self.updateCacheSync()

    if self.state != State.PARSING and not self.parsecache_valid:
        ...
        self.parseConfiguration()
        ...
        for mc in self.multiconfigs:
            (filelist, masked, search) = self.collections[mc].collect_bbfiles(
                self.databuilder.mcdata[mc],
                self.databuilder.mcdata[mc])
```

The important input is `BBFILES`. That variable was assembled earlier while each `layer.conf` file was parsed. Each layer contributed glob patterns that describe where its recipes live.

`collect_bbfiles()` reads those patterns:

**[`lib/bb/cooker.py:1842`](https://git.openembedded.org/bitbake/tree/lib/bb/cooker.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n1842)**
```python
def collect_bbfiles(self, config, eventdata):
    """Collect all available .bb build files"""
    files = (config.getVar("BBFILES") or "").split()
```

This is the first major pivot in the recipe pipeline. `collect_bbfiles()` does not read `pkgs_to_build`. It does not care that the command line target was `demo-image-weston`. It expands every `BBFILES` glob from every enabled layer and finds every available `.bb` file.

That means `demo-image-weston.bb` is discovered because it matches a layer glob from `meta-tegrademo`, not because the target name has been resolved.

---

## Every recipe is parsed into the cache

Once the file list exists, BitBake creates a `CookerParser`:

**[`lib/bb/cooker.py:1675`](https://git.openembedded.org/bitbake/tree/lib/bb/cooker.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n1675)**
```python
self.parser = CookerParser(self, mcfilelist, total_masked)
...
if not self.parser.parse_next():
    bb.server.process.serverlog("Parsing completed")
```

`CookerParser` parses one recipe per call to `parse_next()`. The server idle loop keeps calling `updateCache()` until the parser is done. During this window, BitBake is not executing compile tasks or image tasks. It is reading metadata.

Each recipe parse starts from a copy of the global datastore:

**[`lib/bb/cookerdata.py:541`](https://git.openembedded.org/bitbake/tree/lib/bb/cookerdata.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n541)**
```python
bb_data = self.data.createCopy()
datastores = self._parse_recipe(bb_data, bbfile, appends, '', layername)
```

Then `_parse_recipe()` records recipe context and sends the file to the parser:

**[`lib/bb/cookerdata.py:510`](https://git.openembedded.org/bitbake/tree/lib/bb/cookerdata.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n510)**
```python
def _parse_recipe(bb_data, bbfile, appends, mc, layername):
    bb_data.setVar("__BBMULTICONFIG", mc)
    bb_data.setVar("FILE_LAYERNAME", layername)
    bb_data.setVar("__BB_RECIPE_FILE", bbfile)
    ...
    return bb.parse.handle(bbfile, bb_data)
```

The dispatcher picks the `.bb` handler:

**[`lib/bb/parse/__init__.py:114`](https://git.openembedded.org/bitbake/tree/lib/bb/parse/__init__.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n114)**
```python
def handle(fn, data, include=0, baseconfig=False):
    for h in handlers:
        if h['supports'](fn, data):
            with data.inchistory.include(fn):
                return h['handle'](fn, data, include, baseconfig)
```

The Python recipe handler resolves the file, parses it into statements, evaluates those statements, then finalizes the recipe:

**[`lib/bb/parse/parse_py/BBHandler.py:146`](https://git.openembedded.org/bitbake/tree/lib/bb/parse/parse_py/BBHandler.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n146)**
```python
abs_fn = resolve_file(fn, d)
statements = get_statements(fn, abs_fn, base_name)
...
statements.eval(d)
...
if ext != ".bbclass" and include == 0:
    return ast.multi_finalize(fn, d)
```

This is where assignments, `require`, `inherit`, shell functions, Python functions, anonymous Python blocks, and `addtask` declarations become recipe metadata. The next post zooms into what this produces for `demo-image-weston.bb`.

---

## The cache creates the lookup table

Parsing every recipe would not help target resolution unless the results were indexed. For each parsed recipe, BitBake records `PN`, which is the package name. If the recipe did not set `PN`, BitBake derives it from the filename.

**[`lib/bb/cache.py:94`](https://git.openembedded.org/bitbake/tree/lib/bb/cache.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n94)**
```python
self.pn = self.getvar('PN', metadata) or bb.parse.vars_from_file(filename, metadata)[0]
```

For `demo-image-weston.bb`, the filename fallback produces `PN = "demo-image-weston"`:

**[`lib/bb/parse/__init__.py:151`](https://git.openembedded.org/bitbake/tree/lib/bb/parse/__init__.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n151)**
```python
def vars_from_file(mypkg, d):
    myfile = os.path.splitext(os.path.basename(mypkg))
    parts = myfile[0].split('_')
    ...
    return parts
```

The cache then records both directions:

**[`lib/bb/cache.py:174`](https://git.openembedded.org/bitbake/tree/lib/bb/cache.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n174)**
```python
cachedata.pkg_fn[fn] = self.pn
cachedata.pkg_pn[self.pn].append(fn)

provides = [self.pn]
for provide in self.provides:
    if provide not in provides:
        provides.append(provide)

for provide in provides:
    cachedata.providers[provide].append(fn)
```

For this recipe, the useful result is:

```python
providers["demo-image-weston"] = [
    ".../layers/meta-tegrademo/recipes-demo/images/demo-image-weston.bb"
]
```

This is the join point. The command line target started as a string. The cache turns that string into a provider key that can later resolve to an actual recipe file.

---

## Target resolution happens after parsing

Once parsing finishes, the cooker reaches `State.RUNNING`. On the next idle pass, `runAsyncCommand()` no longer calls `updateCache()`. It finally calls the real `BBCooker.buildTargets()`.

**[`lib/bb/cooker.py:1516`](https://git.openembedded.org/bitbake/tree/lib/bb/cooker.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n1516)**
```python
def buildTargets(self, targets, task):
    self.reset_mtime_caches()
    self.buildSetVars()

    if task is None:
        task = self.configuration.cmd

    if not task.startswith("do_"):
        task = "do_%s" % task

    packages = [target if ':' in target else '%s:%s' % (target, task)
                for target in targets]
    bb.event.fire(bb.event.BuildInit(packages), self.data)

    taskdata, runlist = self.buildTaskData(targets, task, self.configuration.halt)
```

For this build:

```python
targets = ["demo-image-weston"]
task = "build"
task = "do_build"
```

`buildTargets()` is not parsing `demo-image-weston.bb` from scratch. The parse already happened during cache construction. Now BitBake normalizes the task name and asks `buildTaskData()` to resolve the target through the cache.

**[`lib/bb/cooker.py:634`](https://git.openembedded.org/bitbake/tree/lib/bb/cooker.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n634)**
```python
targetlist = self.checkPackages(pkgs_to_build, task)
...
for k in fulltargetlist:
    ...
    taskdata[mc].add_provider(localdata[mc], self.recipecaches[mc], k)
    ...
    fn = taskdata[mc].build_targets[k][0]
    runlist.append([mc, k, ktask, fn])
```

For `k = "demo-image-weston"`, `add_provider()` looks in `dataCache.providers`:

**[`lib/bb/taskdata.py:337`](https://git.openembedded.org/bitbake/tree/lib/bb/taskdata.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n337)**
```python
if not item in dataCache.providers:
    raise bb.providers.NoProvider(item)

all_p = dataCache.providers[item]
eligible, foundUnique = bb.providers.filterProviders(all_p, item, cfgData, dataCache)
...
for fn in eligible:
    self.add_build_target(fn, item)
    self.add_tasks(fn, dataCache)
```

The conceptual runlist entry is now:

```python
[
    "",
    "demo-image-weston",
    "do_build",
    ".../layers/meta-tegrademo/recipes-demo/images/demo-image-weston.bb",
]
```

That runlist is the bridge to task execution. It names one target task, but it does not mean only one task will run. The target task is the starting point for a dependency graph.

---

## Where we are

| Thing | State |
|------|-------|
| Global config datastore | Complete and frozen |
| Recipe files | Discovered through `BBFILES` |
| Recipe metadata | Parsed into `recipecaches` |
| Target string | Resolved through `providers["demo-image-weston"]` |
| Requested task | Normalized from `build` to `do_build` |
| RunQueue input | `demo-image-weston:do_build` |

The key idea is that BitBake parses broadly before it resolves narrowly. The target name does not choose which recipes are parsed. It chooses a provider from the cache after the cache exists. The next post follows the one recipe parse that matters most to this build, `demo-image-weston.bb`.
