---
title: "BitBake startup: the UI handshake and config trigger"
date: 2026-05-12T22:00:00
categories:
  - Jetson Orin Nano
  - Yocto
  - Deep Dive
---

# BitBake startup: the UI handshake and config trigger

You type `bitbake demo-image-weston` and press Enter. By the time the two-process startup completes, the server is blocked in a `select` loop and the UI client is about to enter `knotty.main()`. This post traces what happens in that handshake: how `"demo-image-weston"` got into `configParams` in the first place, what six commands `knotty.main()` sends to the server, and how the very first `getVariable` call silently triggers the entire config-loading pipeline.

If you want the story of how that two-process state was reached, the double-fork and daemonization, see [BitBake startup: the two-process model](./2-bitbake-startup-process-model.md).

<!-- more -->

---

!!! note
    All source links point to BitBake commit [`941cfe7`](https://git.openembedded.org/bitbake/log/?id=941cfe74c19e01e977101e95b09425d21faf844d) (`yocto-6.0_M3`).

!!! warning
    This post is my own trace through the source code. I followed the call chain carefully, but my interpretation of what each part does may be incorrect. If you spot an error, please let me know.

---

## Roadmap

The chain this post covers, from `knotty.main()` entering to the config pipeline firing:

```
1.  knotty.py:474    UI enters knotty.main() → params.updateToServer()
2.  process.py:264   server receives command → runCommand(command, self)
3.  command.py:65    command not in whitelist ["updateConfig","setFeatures","ping"]
                      → init_configdata()
4.  cooker.py:199    init_configdata() : first time → initConfigurationData()
5.  cooker.py:299    initConfigurationData() → CookerDataBuilder + parseBaseConfiguration()
6.  cookerdata.py:274 parseBaseConfiguration() → parseConfigurationFiles(prefiles, postfiles)
```

But before we reach step 1, there is a prerequisite: where does `"demo-image-weston"` actually come from, and how does it end up in the `params` object that `knotty.main()` receives?

---

## Tracing `pkgs_to_build` back to `sys.argv`

The `bin/bitbake` entry point starts with:

**[`bin/bitbake:36`](https://git.openembedded.org/bitbake/tree/bin/bitbake?id=941cfe74c19e01e977101e95b09425d21faf844d#n36)**
```python
sys.exit(bitbake_main(BitBakeConfigParameters(sys.argv),
                      cookerdata.CookerConfiguration()))
```

`BitBakeConfigParameters(sys.argv)` is evaluated first, before `bitbake_main` is called. `sys.argv` is the raw list of strings Python receives from the shell. For `bitbake demo-image-weston`:

```python
sys.argv = ["<path>/bin/bitbake", "demo-image-weston"]
#            argv[0] = the script   argv[1] = target you typed
```

If you had typed `bitbake demo-image-weston core-image-minimal`, `sys.argv` would be:
```python
sys.argv = ["<path>/bin/bitbake", "demo-image-weston", "core-image-minimal"]
```

So `pkgs_to_build` ends up containing every recipe name you typed on the command line. Now let's trace how that list is extracted.

`BitBakeConfigParameters` is a subclass of `ConfigParameters`. Its `__init__` calls `parseCommandLine` immediately:

**[`lib/bb/cookerdata.py:27–31`](https://git.openembedded.org/bitbake/tree/lib/bb/cookerdata.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n27)**
```python
class ConfigParameters(object):
    def __init__(self, argv=None):
        self.options, targets = self.parseCommandLine(argv or sys.argv)
        self.options.pkgs_to_build = targets or []
```

`parseCommandLine` is implemented in `BitBakeConfigParameters` (in `main.py`). It uses Python's `argparse` library, the standard library module for parsing command-line arguments. You define what flags and positional arguments your program accepts, and `argparse` splits `sys.argv` into structured fields automatically:

**[`lib/bb/main.py:314–352`](https://git.openembedded.org/bitbake/tree/lib/bb/main.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n314)**
```python
class BitBakeConfigParameters(cookerdata.ConfigParameters):
    def parseCommandLine(self, argv=sys.argv):
        parser = create_bitbake_parser()                    # defines all accepted flags
        options = parser.parse_intermixed_args(argv[1:])    # parse, skip argv[0] (script name)
        ...
        return options, options.targets                     # targets = positional recipe names
```

`create_bitbake_parser()` is where all the flags are defined. The recipe names you type on the command line are declared as a positional argument that accepts zero or more values:

**[`lib/bb/main.py:135–137`](https://git.openembedded.org/bitbake/tree/lib/bb/main.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n135)**
```python
general_group.add_argument("targets", nargs="*",
                            metavar="recipename/target", ...)
```

`nargs="*"` means "zero or more". When `argparse` sees `["demo-image-weston"]` in `argv[1:]`, a string that does not start with `-`, it goes into `options.targets`. After parsing:

```python
options.targets = ["demo-image-weston"]   # recipe names you typed
options.cmd     = None                    # -c/--cmd flag not passed → stays None
options.force   = False                   # -f not passed
options.debug   = 0                       # -d not passed
# ... etc.
```

`parseCommandLine` returns `(options, options.targets)`. Back in `ConfigParameters.__init__`:

```python
self.options, targets = self.parseCommandLine(...)
self.options.pkgs_to_build = targets or []
# → pkgs_to_build == ["demo-image-weston"]
```

Then every field from `options` is copied onto `self` directly:

```python
for key, val in self.options.__dict__.items():
    setattr(self, key, val)
```

After `__init__` completes: `configParams.pkgs_to_build == ["demo-image-weston"]` and `configParams.cmd == None`.

---

## `cmd` is `None` and will be resolved later

`options.cmd` corresponds to the `-c` / `--cmd` flag, which lets you run a specific task instead of the default. For example:

```sh
bitbake demo-image-weston -c compile
```

would set `options.cmd = "compile"`, and only `do_compile` would run. Without `-c`, `options.cmd` stays `None`. It will be filled in during the UI-server handshake by asking the server for `BB_DEFAULT_TASK`.

---

## `configParams` travels to `knotty.main()`

This `configParams` object, carrying `pkgs_to_build = ["demo-image-weston"]` and `cmd = None`, is passed to `bitbake_main`, which passes it into `knotty.main()`:

**[`lib/bb/main.py:408`](https://git.openembedded.org/bitbake/tree/lib/bb/main.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n408)**
```python
return ui_module.main(server_connection.connection, server_connection.events, configParams)
```

`configParams` arrives in `knotty.main()` as the `params` parameter. The handshake begins.

---

## The six-command handshake

`knotty.main()` sends exactly six commands to the server before the build starts. Let's walk through each one.

---

### Command 1: `updateConfig`

The first thing `knotty.main()` does is push the CLI options (flags like `--force`, `--debug`, etc.) to the server:

**[`lib/bb/ui/knotty.py:474–478`](https://git.openembedded.org/bitbake/tree/lib/bb/ui/knotty.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n474)**
```python
def main(server, eventHandler, params, tf=TerminalFilter):
    if not params.observe_only:
        params.updateToServer(server, os.environ.copy())
```

`updateToServer` sends one `updateConfig` command carrying all CLI options:

**[`lib/bb/cookerdata.py:59–76`](https://git.openembedded.org/bitbake/tree/lib/bb/cookerdata.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n59)**
```python
def updateToServer(self, server, environment):
    options = {}
    for o in ["halt", "force", "invalidate_stamp", "dry_run", ...]:
        options[o] = getattr(self.options, o)
    ret, error = server.runCommand(["updateConfig", options, environment, sys.argv])
```

`updateConfig` is on the server's whitelist of commands that do **not** trigger config loading. The server stores the options and returns immediately.

---

### Command 2: `getVariable` (the config trigger)

After `updateToServer`, `knotty.main()` calls `_log_settings_from_server()`. This sends several `getVariable` commands to fetch logging configuration: `BBINCLUDELOGS`, `BB_CONSOLELOG`, and a few more.

`getVariable` is **not** on the whitelist. The first `getVariable` is the trigger: when the server receives it, the entire config-loading pipeline fires. The remaining `getVariable` calls in `_log_settings_from_server()` are no-ops because the config is already loaded by then.

We will trace exactly what happens on the server side in the next section. For now, keep following knotty's side.

---

### Command 3: `setEventMask`

After logging settings, knotty tells the server which event types to stream back to the UI:

**[`lib/bb/ui/knotty.py:673`](https://git.openembedded.org/bitbake/tree/lib/bb/ui/knotty.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n673)**
```python
server.runCommand(["setEventMask", server.getEventHandle(),
                   llevel, debug_domains, _evt_list])
```

---

### Commands 4 and 5: resolving `cmd` via `updateFromServer`

`knotty.main()` then calls `params.updateFromServer(server)`. Here, `cmd` is finally resolved: since it is still `None`, `updateFromServer` asks the server for `BB_DEFAULT_TASK`:

**[`lib/bb/cookerdata.py:42–57`](https://git.openembedded.org/bitbake/tree/lib/bb/cookerdata.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n42)**
```python
def updateFromServer(self, server):
    if not self.options.cmd:                           # cmd is None → ask the server
        defaulttask, error = server.runCommand(["getVariable", "BB_DEFAULT_TASK"])
        self.options.cmd = defaulttask or "build"      # → "build"
    server.runCommand(["setConfig", "cmd", self.options.cmd])

    if not self.options.pkgs_to_build:                 # already ["demo-image-weston"], skipped
        bbpkgs, error = server.runCommand(["getVariable", "BBTARGETS"])
        ...
```

By now `BB_DEFAULT_TASK` is available in `self.data` (loaded back when command 2 arrived), so `getVariable BB_DEFAULT_TASK` returns `"build"`. After `updateFromServer`:

```python
params.options.cmd = "build"
```

`setConfig` (command 5) pushes that back to the server. Note that `pkgs_to_build` is already `["demo-image-weston"]`, so the `BBTARGETS` fallback is skipped entirely.

---

### Command 6: `buildTargets`

`params.parseActions()` assembles the final server command:

**[`lib/bb/cookerdata.py:107–109`](https://git.openembedded.org/bitbake/tree/lib/bb/cookerdata.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n107)**
```python
if self.options.pkgs_to_build:
    action['action'] = ["buildTargets", self.options.pkgs_to_build, self.options.cmd]
    # → ["buildTargets", ["demo-image-weston"], "build"]
```

`knotty.main()` sends it:

**[`lib/bb/ui/knotty.py:702`](https://git.openembedded.org/bitbake/tree/lib/bb/ui/knotty.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n702)**
```python
ret, error = server.runCommand(cmdline['action'])
# cmdline['action'] == ["buildTargets", ["demo-image-weston"], "build"]
```

---

## Full command sequence

```
1. updateConfig        ← CLI options (force, debug, etc.)          updateToServer()
2. getVariable         ← BBINCLUDELOGS, BB_CONSOLELOG, etc.        _log_settings_from_server()
                          ↑ THIS IS THE CONFIG TRIGGER (first non-whitelisted command)
3. setEventMask        ← which events to stream back               knotty.py:673
4. getVariable         ← BB_DEFAULT_TASK → "build"                 updateFromServer()
5. setConfig cmd build ← push task name to server                  updateFromServer()
6. buildTargets        ← ["demo-image-weston"], "build"            knotty.py:702
```

---

## What happens on the server when command 2 arrives

Now let's follow the server's side from the moment the first `getVariable` hits its socket.

The server is asleep in `select`. The socket becomes readable. `main()` wakes up, reads the command off the socket, and dispatches it:

**[`lib/bb/server/process.py:264–279`](https://git.openembedded.org/bitbake/tree/lib/bb/server/process.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n264)**
```python
if self.command_channel in ready:
    try:
        command = self.command_channel.get()
    except EOFError:
        ready = []
        disconnect_client(self, fds)
        continue
    ...
    try:
        serverlog("Running command %s" % command)
        reply = self.cooker.command.runCommand(command, self)
        serverlog("Sending reply %s" % repr(reply))
        self.command_channel_reply.send(reply)
```

`self.cooker.command.runCommand(command, self)` is the dispatch point. Inside `runCommand`, the first check is against the whitelist:

**[`lib/bb/command.py:65–71`](https://git.openembedded.org/bitbake/tree/lib/bb/command.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n65)**
```python
if command not in ["updateConfig", "setFeatures", "ping"]:
    try:
        self.cooker.init_configdata()
```

The whitelist is exactly three commands: `updateConfig`, `setFeatures`, and `ping`. Everything else (`getVariable`, `setConfig`, `setEventMask`, `buildTargets`) goes through `init_configdata()` first. `getVariable` is not on the list, so `init_configdata()` is called.

---

## `init_configdata`: the one-shot guard

`init_configdata` only does real work the first time, when `self.data` does not yet exist:

**[`lib/bb/cooker.py:199–203`](https://git.openembedded.org/bitbake/tree/lib/bb/cooker.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n199)**
```python
def init_configdata(self):
    if not hasattr(self, "data"):
        self.initConfigurationData()
```

The first time (triggered by command 2), `self.data` does not exist, so `initConfigurationData()` runs. Every subsequent command (`setEventMask`, `getVariable BB_DEFAULT_TASK`, `setConfig`, `buildTargets`) also calls `init_configdata()`, but `self.data` now exists, so it is a no-op.

---

## `initConfigurationData`: kicking off the config pipeline

`initConfigurationData()` creates the data builder and triggers the actual config parse:

**[`lib/bb/cooker.py:299–301`](https://git.openembedded.org/bitbake/tree/lib/bb/cooker.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n299)**
```python
self.databuilder = bb.cookerdata.CookerDataBuilder(self.configuration, False)
self.databuilder.parseBaseConfiguration()
self.data = self.databuilder.data
```

`CookerDataBuilder.__init__` creates `basedata`, a blank `DataSmart` object (BitBake's variable store) seeded only with filtered shell environment variables (`HOME`, `PATH`, `USER`, etc.). No `.conf` file has been touched yet.

`parseBaseConfiguration()` then takes that blank `basedata` and builds the full variable namespace by reading all the `.conf` files:

**[`lib/bb/cookerdata.py:274–277`](https://git.openembedded.org/bitbake/tree/lib/bb/cookerdata.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n274)**
```python
def parseBaseConfiguration(self, worker=False):
    mcdata = {}
    try:
        self.data = self.parseConfigurationFiles(self.prefiles, self.postfiles)
```

`parseConfigurationFiles` is where `bblayers.conf`, each layer's `layer.conf`, and `conf/bitbake.conf` (which pulls in `local.conf`, the machine conf, and the distro conf) are actually read.

---

## The full chain inside one `runCommand` call

All of this, from the server waking up to `parseConfigurationFiles` returning, happens synchronously inside the single `runCommand("getVariable", "BBINCLUDELOGS")` call. The UI client is blocked, waiting for the reply. Here is the complete chain:

```
runCommand receives "getVariable BBINCLUDELOGS"
  │
  ├── "getVariable" not in whitelist → init_configdata()
  │     └── self.data doesn't exist yet → initConfigurationData()
  │           ├── CookerDataBuilder(self.configuration)
  │           │     └── basedata = DataSmart() + filtered env vars
  │           └── databuilder.parseBaseConfiguration()
  │                 └── parseConfigurationFiles(prefiles, postfiles)
  │                       ├── bblayers.conf     → BBLAYERS, BBPATH
  │                       ├── each layer's layer.conf → BBFILES, BBFILE_COLLECTIONS
  │                       └── conf/bitbake.conf → local.conf, machine conf, distro conf
  │     init_configdata() returns — self.data now fully populated
  │
  └── execute getVariable handler → reads "BBINCLUDELOGS" from self.data → returns value
```

The UI receives the `BBINCLUDELOGS` value and continues with the rest of the handshake, completely unaware that the entire config pipeline just ran inside that one call.

---

## What is ready after the handshake

After the config trigger fires and the first `getVariable` returns:

- `self.data` exists and is fully populated: a `DataSmart` holding every variable defined in every `.conf` file across all layers.
- `BB_DEFAULT_TASK`, `BBINCLUDELOGS`, `BB_CONSOLELOG`, and all other global variables are queryable.
- Every subsequent handshake command (`setEventMask`, `getVariable BB_DEFAULT_TASK`, `setConfig`, `buildTargets`) is fast: it reads or sets a single value with no config loading involved.

What `parseConfigurationFiles` actually does, how it walks `BBLAYERS`, reads each `layer.conf`, and assembles the full variable namespace, is a story of its own.
