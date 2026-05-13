---
title: "BitBake startup: the two-process model"
date: 2026-05-12T21:00:00
categories:
  - Jetson Orin Nano
  - Yocto
  - Deep Dive
---

# BitBake startup: the two-process model

You type `bitbake demo-image-weston` and press Enter. Before the first recipe is parsed, before any `.conf` file is touched, something interesting happens at the OS level: the single process you launched splits in two, the second one daemonizes itself, and the two processes spend the rest of the build communicating over a Unix socket.

This post traces that startup sequence from the shell command down to the server's `select` loop, with every claim linked to the exact source line.

<!-- more -->

---

!!! note
    All source links point to BitBake commit [`941cfe7`](https://git.openembedded.org/bitbake/log/?id=941cfe74c19e01e977101e95b09425d21faf844d) (`yocto-6.0_M3`).

!!! warning
    This post is my own trace through the source code. I followed the call chain carefully, but my interpretation of what each part does may be incorrect. If you spot an error, please let me know.

---

## Roadmap

```
1.  main.py:470      BitBakeServer.__init__()
2.  process.py:546   createDaemon(self._startServer, logfile)
                      → double-fork: original process returns as UI client
                      → grandchild (2nd fork) is the detached daemon
3.  process.py:598   _startServer() → os.execl(bitbake-server)
                      → grandchild replaces itself with the bitbake-server binary
4.  bitbake-server:54  execServer(...)
5.  process.py:630   ProcessServer + BBCooker.__init__()  (no config load yet)
6.  process.py:167   server.run() → main() → select() loop, waiting for UI
```

---

## The entry point

The `bitbake` binary on your `PATH` comes from `source setup-env`, which prepends `repos/bitbake/bin/` to `PATH`. The entry point is two lines:

**[`bin/bitbake:36`](https://git.openembedded.org/bitbake/tree/bin/bitbake?id=941cfe74c19e01e977101e95b09425d21faf844d#n36)**
```python
sys.exit(bitbake_main(BitBakeConfigParameters(sys.argv),
                      cookerdata.CookerConfiguration()))
```

`BitBakeConfigParameters(sys.argv)` parses the command line immediately, extracting `"demo-image-weston"` from `sys.argv` and storing it as `configParams.pkgs_to_build`. By the time `bitbake_main` runs, the build target is already known.

`bitbake_main` lives in `lib/bb/main.py`. Its first action:

**[`lib/bb/main.py:390`](https://git.openembedded.org/bitbake/tree/lib/bb/main.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n390)**
```python
server_connection, ui_module = setup_bitbake(configParams)
```

`setup_bitbake` is where the server process is born.

---

## Spawning the server

Inside `setup_bitbake`, a `BitBakeServer` object is created:

**[`lib/bb/main.py:467–470`](https://git.openembedded.org/bitbake/tree/lib/bb/main.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n467)**
```python
logger.info("Starting bitbake server...")
server = bb.server.process.BitBakeServer(
    lock, sockname, featureset,
    configParams.server_timeout, ...)
```

Inside `BitBakeServer.__init__`, `createDaemon` is called:

**[`lib/bb/server/process.py:546–548`](https://git.openembedded.org/bitbake/tree/lib/bb/server/process.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n546)**
```python
startdatetime = datetime.datetime.now()
bb.daemonize.createDaemon(self._startServer, logfile)
self.bitbake_lock.close()
```

`createDaemon` is where the process splits in two. To understand what it does and why it does it the way it does, we need a short detour into how Unix manages processes.

---

## Unix background: sessions, process groups, and controlling terminals

A Unix process lives inside a three-level hierarchy:

```
session
  └── process group
        └── process
```

- **Process group**: a collection of related processes, typically the commands in a shell pipeline. Every process has a PGID (process group ID).
- **Session**: a collection of process groups, created by `setsid()`. The process that calls `setsid()` becomes the **session leader** of the new session.
- **Controlling terminal**: your shell window or SSH connection is attached to a session. The kernel uses this attachment in two ways:
    - When the terminal closes, it sends `SIGHUP` to the session leader and all processes in the session.
    - Only a session leader can acquire a new controlling terminal by opening a tty device.

The goal of daemonization is to sever both links permanently: get the server into a different session from your terminal, and ensure it can never re-acquire a controlling terminal.

A single `fork()` is not enough: the child inherits the same session as the parent. The child can call `setsid()` to create a new session, but that makes it the session leader. Session leaders can re-acquire controlling terminals, so you would be in a new session but still vulnerable.

The solution is to fork **twice**.

---

## The double-fork

[`lib/bb/daemonize.py`](https://git.openembedded.org/bitbake/tree/lib/bb/daemonize.py?id=941cfe74c19e01e977101e95b09425d21faf844d) performs the double-fork. Here is the process tree and what each step achieves:

```
original process (bitbake UI)
    │
    ├── os.fork() ──────────────────── child 1                   daemonize.py:40
    │       │                               │
    │   waitpid(child1)              os.setsid()                 daemonize.py:48
    │   then returns                  (new session, child 1 is session leader)
    │   (stays as UI client)                │
    │                                os.fork() ──── child 2 (grandchild)   daemonize.py:58
    │                                       │                   │
    │                                  _exit(0)          _startServer()    daemonize.py:94
    │                                  (dies)            → os.execl(bitbake-server)
    │                                                    (the server daemon)
```

!!! note
    `os.fork()` return value: `0` means you are the **child**, non-zero means you are the **parent** (the value is the child's PID). Both processes continue from the same line after the fork. The return value is the only way each side knows its role.

- **First fork:** child 1 is born with its own PID. The original process calls `waitpid` and blocks until child 1 exits, then returns here and continues as the UI client.
- **`os.setsid()`** ([`daemonize.py:48`](https://git.openembedded.org/bitbake/tree/lib/bb/daemonize.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n48)): child 1 creates a brand-new session with no controlling terminal. Child 1 is now the session leader of that session.
- **Second fork:** child 1 forks again and immediately calls `_exit(0)`. The grandchild (child 2) inherits the new session but is **not** the session leader. Only the process that called `setsid()` holds that role. On Linux, only a session leader can acquire a controlling terminal by opening a tty device. Since the grandchild is never the session leader, it can never accidentally pick up a terminal no matter what files it opens later.
- Because child 1 exits, the grandchild becomes an orphan. The kernel re-parents orphaned processes to PID 1 (systemd/init). This is the canonical "fully detached daemon" state.

The original process unblocks from `waitpid` (child 1 has exited) and returns from `createDaemon` as the UI client. The grandchild is now running in the background as a fully detached daemon.

---

## The grandchild becomes the server

The grandchild runs `_startServer()`. Rather than continuing as a Python process that is the grandchild, it `exec`s `bin/bitbake-server`, completely replacing its memory image with a fresh Python interpreter running the server script:

**[`lib/bb/server/process.py:598–603`](https://git.openembedded.org/bitbake/tree/lib/bb/server/process.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n598)**
```python
def _startServer(self):
    os.close(self.readypipe)
    os.set_inheritable(self.bitbake_lock.fileno(), True)
    os.set_inheritable(self.readypipein, True)
    serverscript = os.path.realpath(
        os.path.dirname(__file__) + "/../../../bin/bitbake-server")
    os.execl(sys.executable, sys.executable, serverscript, "decafbad",
             str(self.bitbake_lock.fileno()), str(self.readypipein), self.logfile,
             self.bitbake_lock.name, self.sockname, str(self.server_timeout or 0),
             str(list(self.profile)), str(self.xmlrpcinterface[0]),
             str(self.xmlrpcinterface[1]))
```

`os.execl` passes the lock file descriptor, ready-pipe fd, socket path, and other configuration as command-line arguments to `bitbake-server`. The grandchild's PID is unchanged. Only its code is replaced.

`bin/bitbake-server` immediately calls:

**[`bin/bitbake-server:54`](https://git.openembedded.org/bitbake/tree/bin/bitbake-server?id=941cfe74c19e01e977101e95b09425d21faf844d#n54)**
```python
bb.server.process.execServer(lockfd, readypipeinfd, lockname, sockname,
                              timeout, xmlrpcinterface, profile)
```

We will follow `execServer` in a moment. First, let's see what the original process is doing in parallel.

---

## Meanwhile: the original process becomes the UI client

After `createDaemon` returns (once `waitpid` unblocks with child 1 gone), the original process is back inside `setup_bitbake`. It does two things before returning.

**1. Load the UI module.** `configParams.ui` defaults to `'knotty'` (the terminal UI):

**[`lib/bb/main.py:435–436`](https://git.openembedded.org/bitbake/tree/lib/bb/main.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n435)**
```python
ui_module = import_extension_module(bb.ui, configParams.ui, 'main')
# configParams.ui == 'knotty'  →  ui_module = bb.ui.knotty
```

**2. Connect to the server.** The grandchild, now running as `bitbake-server`, creates a Unix socket at `bitbake.sock`. The original process connects to it:

**[`lib/bb/main.py:491`](https://git.openembedded.org/bitbake/tree/lib/bb/main.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n491)**
```python
server_connection = bb.server.process.connectProcessServer(sockname, featureset)
```

`setup_bitbake` returns both `server_connection` and `ui_module`. Back in `bitbake_main`, the original process calls `knotty.main()` as a plain function call in the same Python process that has been running since `bin/bitbake` started. No new process, no exec:

**[`lib/bb/main.py:408`](https://git.openembedded.org/bitbake/tree/lib/bb/main.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n408)**
```python
return ui_module.main(server_connection.connection, server_connection.events, configParams)
```

The two-process picture after the double-fork is now complete:

```
original process  →  knotty.main()
                      (UI loop: sends commands over the socket)
                              │
                         bitbake.sock
                              │
grandchild        →  execServer()
                      (BBCooker + select loop: receives commands)
```

The two processes share no memory, no Python objects, nothing. They communicate only through messages over `bitbake.sock`.

---

## Inside `execServer`: creating the cooker

Back on the server side, `execServer` creates a `ProcessServer` and a `BBCooker`, then starts the server loop:

**[`lib/bb/server/process.py:630–643`](https://git.openembedded.org/bitbake/tree/lib/bb/server/process.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n630)**
```python
server = ProcessServer(bitbake_lock, lockname, sock, sockname,
                       server_timeout, xmlrpcinterface)
writer = ConnectionWriter(readypipeinfd)
try:
    featureset = []
    cooker = bb.cooker.BBCooker(featureset, server)
    cooker.configuration.profile = profile
except bb.BBHandledException:
    return None
writer.send("r")
writer.close()
server.cooker = cooker
serverlog("Started bitbake server pid %d" % os.getpid())

server.run()
```

`writer.send("r")` signals through the ready-pipe back to the original process that the server is ready. This is how `connectProcessServer` in the UI knows when to stop waiting and proceed.

`BBCooker(featureset, server)` runs `__init__`. The key thing to notice is what it does **not** do:

**[`lib/bb/cooker.py:137–191`](https://git.openembedded.org/bitbake/tree/lib/bb/cooker.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n137)**
```python
def __init__(self, featureSet=None, server=None):
    self.recipecaches = None
    self.baseconfig_valid = False
    ...
    self.configuration = bb.cookerdata.CookerConfiguration()
    ...
    self.command = bb.command.Command(self, self.process_server)
    self.state = State.INITIAL
    # config load is deliberately deferred until the first command
```

No `.conf` file is opened. No `bblayers.conf`, no `bitbake.conf`, nothing. `BBCooker.__init__` only allocates in-memory state. Config loading is deferred until the first real command arrives from the UI. This is intentional: if the UI only queries a variable without triggering a build, there is no reason to parse every layer's `.conf` files up front.

---

## `server.run()` → `main()` → the `select` loop

After `BBCooker.__init__` returns, `execServer` calls `server.run()`. `run()` does a small amount of housekeeping: it optionally starts an XML-RPC server if `--xmlrpc` was passed, writes the server's PID into the lock file, then calls `main()`:

**[`lib/bb/server/process.py:121–140`](https://git.openembedded.org/bitbake/tree/lib/bb/server/process.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n121)**
```python
def run(self):
    if self.xmlrpcinterface[0]:
        self.xmlrpc = bb.server.xmlrpcserver.BitBakeXMLRPCServer(...)

    self.bitbake_lock.write("%s\n" % (os.getpid()))
    self.bitbake_lock.flush()

    return bb.utils.profile_function(
        "main" in self.cooker.configuration.profile,
        self.main, "profile-mainloop.log")
```

`profile_function` is a thin wrapper: if the `profile` option contains `"main"`, it wraps the call in Python's `cProfile` and writes timing data to `profile-mainloop.log`. Otherwise it calls `self.main()` directly. Either way, `main()` runs.

**[`lib/bb/server/process.py:167–169`](https://git.openembedded.org/bitbake/tree/lib/bb/server/process.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n167)**
```python
def main(self):
    self.cooker.pre_serve()
    # ... then the select loop
```

`pre_serve()` is a hook for one-time setup before the event loop begins. In stock BitBake it does nothing:

**[`lib/bb/cooker.py:1746–1747`](https://git.openembedded.org/bitbake/tree/lib/bb/cooker.py?id=941cfe74c19e01e977101e95b09425d21faf844d#n1746)**
```python
def pre_serve(self):
    return
```

After `pre_serve`, `main()` enters the `select` loop. `select` is a standard Unix system call that lets a process watch multiple file descriptors at once and sleep until one has data ready. The server is not spinning or burning CPU. It is genuinely asleep, parked by the kernel, until a message arrives.

The file descriptors it watches are:

- **the command channel**: the Unix socket where the UI sends commands
- **the UI connection socket**: where new UI client connections initially arrive
- **a self-pipe**: used internally to wake the loop for clean shutdown

The server is now fully running but completely idle: no files open, no config parsed, just waiting.

---

## Why the server survives after the terminal closes

This is a direct consequence of the double-fork. The grandchild was placed in a different session from your terminal entirely. When the terminal closes:

- The kernel sends `SIGHUP` to the processes in the terminal's session.
- The server is in a different session, so `SIGHUP` never reaches it.
- The UI client (`knotty.main()`) is still in the original process and session, so it does exit when the terminal closes. But that is only one side of the socket going away.
- The server process keeps running, keeps building.

This is intentional design. If your SSH connection drops mid-build, the server continues. You can reconnect a new UI to the still-running server to check progress. `bitbake --server-only` and `--remote-server` exploit this design explicitly.

---

## Ctrl-C still stops the build, but through the socket not a signal

The double-fork put the server outside the terminal's process group. The kernel's automatic `SIGINT` broadcast on Ctrl-C does not reach it. Termination is always sent explicitly through the socket:

```
Ctrl-C
  → SIGINT → knotty.py signal handler
  → server.runCommand(["stateShutdown"])
  → server receives the shutdown command, stops cleanly
```

---

## What if the UI crashes without sending shutdown?

The server has a `--server-timeout` (default 60 seconds). If no UI connects within that window, or if the last connected UI disconnects without sending `stateShutdown`, the server starts a countdown. When it expires, the server exits on its own and does not hang indefinitely.

---

## Where we are

Two processes are now running:

| Process | Entry point | State |
|---------|-------------|-------|
| Original (`bitbake`) | `knotty.main()` | About to send commands over the socket |
| Grandchild (`bitbake-server`) | `main()` | Blocked in `select`, no config loaded |

The server is waiting. The UI client has `configParams` in hand, carrying `["demo-image-weston"]` as the build target. The two-process split is complete. Everything from here is message passing over the socket.

[BitBake startup: the UI handshake and config trigger](./3-bitbake-startup-handshake.md) traces what happens next: how `knotty.main()` handshakes with the server and which command silently triggers the entire config-loading pipeline.
