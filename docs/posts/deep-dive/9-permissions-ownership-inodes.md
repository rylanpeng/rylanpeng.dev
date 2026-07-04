---
title: What is a file? Permissions, ownership, and the inode model
date: 2026-07-04
categories:
  - Linux Fundamentals
  - Deep Dive
---

# What is a file? Permissions, ownership, and the inode model

A walkthrough of permission bits, ownership, and the inode model: the things you see constantly in Yocto recipes and everyday Linux commands but rarely see explained end to end. By the end you can read any permission number on sight, know what `-o root -g root` does, and understand what a file actually is on disk (filename → inode → data blocks), which is also the key to why the atomic rename trick works.

<!-- more -->

---

You'll see permission bits like these constantly, in Yocto recipes:

```bitbake
install -d -m 0755 ${D}${bindir}
install -m 0644 myconfig.conf ${D}${sysconfdir}/myconfig.conf
install -m 0600 secret.key ${D}${sysconfdir}/secret.key
```

and in everyday Linux commands:

```sh
install -o root -g root -m 0755 startup.sh /usr/bin/startup.sh
chown root:root /etc/shadow
chmod 600 ~/.ssh/id_rsa
```

---

## Part 0: Reading a real `ls -l` line

Before the numbers, here's how to decode the line `ls -l` prints. Every field you'll learn about below appears in it. Take a generic line:

```
-rw-r--r--  1 alice  staff  4096 Jan  1 12:00 report.txt
│└───┬───┘  │ └─┬─┘  └─┬─┘  └─┬─┘ └────┬────┘ └────┬────┘
│    │      │   │      │      │        │           └ name
│    │      │   │      │      │        └ last-modified time
│    │      │   │      │      └ size in bytes
│    │      │   │      └ GROUP owner   → "staff"
│    │      │   └ USER (owner)         → "alice"
│    │      └ link count               → 1
│    └ the 9 permission bits           → rw-r--r--  (= 0644, see Part 1)
└ file TYPE                            → - = regular file
```

The fields, left to right: **type · permissions · link count · owner · group · size · timestamp · name**. The next sections explain each one.

### 0.1 The first character: file type

The very first character is **not** a permission, it's *what kind of thing this is*:

| Char | Type |
|------|------|
| `-` | regular file |
| `d` | **directory** |
| `l` | symbolic link |
| `c` | character device (e.g. `/dev/tty`) |
| `b` | block device (e.g. `/dev/sda`) |
| `s` | socket |
| `p` | named pipe (FIFO) |

So a leading `d` (as in `drwxr-xr-x`) means **"this is a directory."** The `-` in our `-rw-r--r--` example means a plain regular file. Either way, the remaining 9 characters are the permissions from Part 1.

### 0.2 The number: link count

This field is the **hard-link count**: how many directory entries point to this inode (see Part 3 for what an inode is).

- For a **regular file**, this is usually `1` (one name → one inode), as in our example. It goes up if you make hard links to it.
- For a **directory**, it starts at `2` and up. Why 2? Because a directory is referred to by *two* names the moment it's created:
  1. its own name in the parent (e.g. `mydir`), and
  2. the `.` entry **inside** itself (`mydir/.`).

  Every **subdirectory** you add bumps it by one more, because each child contains a `..` entry pointing back up to this directory. So a directory's link count is roughly `2 + (number of subdirectories)`.

### 0.3 The two names: owner and group

```
... alice  staff ...
    └─┬─┘  └─┬─┘
      │      └ GROUP owner  (the file's group)
      └ USER owner          (the person who owns the file)
```

Here the file is owned by user `alice`, group `staff`. The two names are often *identical* because Linux, by default, gives each user a **personal group of the same name** (user `bob` → group `bob`), so you'll frequently see lines like `bob bob`. They can differ too, e.g. `root docker` (owned by root, group docker), which is exactly the next topic.

---

## Part 1: What those numbers mean (0755, 600, 0664, ...)

### 1.1 The three "who" groups

Every file has permissions for **three classes of user**:

| Class | Who |
|-------|-----|
| **user** (`u`) | the owner of the file |
| **group** (`g`) | members of the file's group |
| **other** (`o`) | everyone else |

### 1.2 The three permission bits

For each class, there are three permissions:

| Bit | Symbol | Value | On a file | On a directory |
|-----|--------|-------|-----------|----------------|
| read    | `r` | **4** | read the contents | list the entries (`ls`) |
| write   | `w` | **2** | modify the contents | create/delete/rename entries inside |
| execute | `x` | **1** | run it as a program | `cd` into it / access entries by name |

The values **4, 2, 1** are powers of two, so any combination adds up to a **unique number 0-7**:

```
rwx = 4+2+1 = 7
rw- = 4+2+0 = 6
r-x = 4+0+1 = 5
r-- = 4+0+0 = 4
-wx = 0+2+1 = 3
-w- = 0+2+0 = 2
--x = 0+0+1 = 1
--- = 0+0+0 = 0
```

### 1.3 Reading a 3-digit number

Each digit is one class, in the order **user, group, other**:

```
  7 5 5
  │ │ └── other : 5 = r-x  (read + execute)
  │ └──── group : 5 = r-x  (read + execute)
  └────── user  : 7 = rwx  (read + write + execute)
```

So `0755` = `rwxr-xr-x` = "owner can do everything, everyone else can read & run but not modify." This is the standard for **executables and directories**.

### 1.4 The most common modes you'll actually see

| Mode | Symbolic | Meaning | Typical use |
|------|----------|---------|-------------|
| `0755` | `rwxr-xr-x` | owner full, others read+exec | scripts, binaries, directories |
| `0644` | `rw-r--r--` | owner read/write, others read-only | normal config/data files |
| `0664` | `rw-rw-r--` | owner+group read/write, others read | group-shared files |
| `0600` | `rw-------` | owner read/write, **nobody else** | secrets, SSH keys, `/etc/shadow`-ish |
| `0700` | `rwx------` | owner everything, nobody else | private directories (`~/.ssh`) |
| `0777` | `rwxrwxrwx` | everyone everything | almost never, a red flag |

!!! note "0664 vs 0644"
    The only difference is the middle digit (group): `6 = rw-` lets the group write, `4 = r--` makes it read-only for the group.

### 1.5 What's the leading `0`?

`0755` and `755` are the **same** permissions. The leading digit is a **fourth permission digit** for special bits, and it's usually `0`:

| Leading bit | Value | Name | Effect |
|-------------|-------|------|--------|
| setuid | **4** | `04755` | run with the **owner's** identity (e.g. `passwd` runs as root) |
| setgid | **2** | `02755` | run with the **group's** identity, and on a directory new files inherit the directory's group |
| sticky | **1** | `01777` | in a shared dir, only the file's owner can delete it (e.g. `/tmp`) |

So writing `0755` is just being explicit that there are **no** special bits. You see the 4-digit form in Yocto recipes because `install -m` wants it unambiguous.

### 1.6 Symbolic mode (the other way to write it)

`chmod` also accepts letters, which is often clearer for *changing* one bit:

```sh
chmod u+x script.sh      # add execute for the owner
chmod go-w file          # remove write for group and other
chmod a+r file           # add read for all (a = u+g+o)
chmod u=rw,go=r file     # set exactly: 0644
```

---

## Part 2: Ownership and what `-o root -g root` means

Permissions answer *"what can each class do?"*. Ownership answers *"who is the owner, and what is the group?"*.

Every file stores **two** ownership fields:

- an **owner** (a user, e.g. `root`, `alice`)
- a **group** (e.g. `root`, `users`, `dialout`)

### 2.1 In the `install` command

```sh
install -o root -g root -m 0755 startup.sh /usr/bin/startup.sh
         └──┬──┘ └──┬──┘ └──┬──┘
            │       │       └── mode (permissions) = rwxr-xr-x
            │       └────────── -g : set the GROUP owner to "root"
            └────────────────── -o : set the USER  owner to "root"
```

So `-o root -g root` means **"after copying, make the new file owned by user `root` and group `root`."** Without it, the file would be owned by whoever ran the command.

This matters in Yocto / image building because the **build runs as a normal user**, but files in the final image must be owned by `root` (or some service account). `install -o ... -g ...` sets the *target* ownership regardless of who is building. The equivalent after-the-fact command is:

```sh
chown root:root /usr/bin/startup.sh    # owner:group, same as -o root -g root
chgrp dialout   /dev/ttyUSB0           # change only the group
```

!!! note
    In plain shell, only **root** can give a file away to another user. In a Yocto recipe, `install` records the intended ownership into the image metadata, so the packaged file ends up `root:root` even though no real `chown` happened during the build.

### 2.2 Real example: running docker without sudo (the group trick)

After installing Docker, the step you're thinking of is:

```sh
sudo usermod -aG docker $USER
# then log out and back in (or run: newgrp docker)
```

**`-aG docker`** = **a**ppend the user to the **G**roup `docker`. This is a pure *group ownership* concept. Here's the whole mechanism:

Docker talks to its daemon through a socket file. Look at who owns it:

```
$ ls -l /var/run/docker.sock
srw-rw---- 1 root docker 0 Jan  1 12:00 /var/run/docker.sock
│└──┬───┘    └─┬─┘ └─┬──┘
│   │          │     └ GROUP = docker
│   │          └ USER  = root
│   └ perms rw-rw---- (0660): owner rw, group rw, OTHER nothing
└ type s = socket
```

Read that with Part 1 + Part 0:

- **owner** is `root`, with `rw-`.
- **group** is `docker`, with `rw-`.
- **other** is `---` → anyone *not* root and *not* in the `docker` group gets **nothing**, so they must use `sudo`.

So `usermod -aG docker $USER` makes you a **member of the `docker` group**. Now you fall into the *group* class for that socket, which has `rw-`, so you can use `docker` directly without `sudo`. You didn't change the file's permissions at all. You changed **which class you belong to** for that file.

!!! warning
    This effectively grants root-equivalent power (the docker daemon runs as root), which is why it's a deliberate opt-in and not the default.

Use the `-a` (append)! `sudo usermod -G docker $USER` *without* `-a` would **replace** all your other groups with just `docker`, a common foot-gun.

Check your groups with:

```sh
groups          # lists every group you belong to
id              # shows uid, gid, and all group memberships
```

### 2.3 Putting Part 1 + Part 2 together

```sh
install -o root -g root -m 0600 secret.key ${D}/etc/secret.key
```

reads as: *"Place `secret.key` at `/etc/secret.key`, owned by `root:root`, with mode `rw-------`: only root can read or write it, nobody else can even read it."* That is exactly how you ship a private key safely.

---

## Part 3: What a file actually is (filename → inode → data blocks)

Now the deeper model. This is what makes "atomic rename" make sense.

A file is **not** a single sequential thing. It's a set of references in three layers:

```
filename  →  inode  →  data blocks
 (label)     (identity   (the actual
              + metadata)  bytes)
```

### 3.1 Layer A: filename (the directory entry)

A filename is **just a label stored in a directory**. A directory does *not* contain file data. It only maps:

```
name → inode number
```

For example, a directory might hold:

```
data.dat  →  inode 1001
data.tmp  →  inode 2002
```

So **filenames are pointers to inodes**. Nothing more.

### 3.2 Layer B: inode (the file's real identity)

The inode is the "real file object." It contains everything *about* the file **except its name**:

- permissions (the `0755` from Part 1)
- ownership (the `root:root` from Part 2)
- timestamps, file size
- **pointers to the data blocks**

```
inode 1001  →  [block A, block C, block D]
inode 2002  →  [block B, block E]
```

Think of the inode as the file's **index card**: it says where the data lives, but it does not know its own name.

!!! note
    An inode is an **ID for a file**, not an ID for a block. It *contains* pointers to many blocks.

### 3.3 Layer C: data blocks (the bytes)

These are raw chunks of bytes on disk:

```
Block A: bytes    0-4095
Block B: bytes 4096-8191
...
```

### 3.4 The relationship is a two-step lookup, not a chain

```
directory entry            "data.dat"
       ↓  (look up name → inode number)
     inode                 inode 1001
       ↓  (read its block pointers)
  block list   →   [block A, block B, block C, ...]
```

It's **hierarchical lookup**, not a sequential `A → B → C` pipeline.

### 3.5 Can two names share one inode? (hard links)

Yes, this is exactly what a **hard link** is:

```
file1  →  inode 1001
file2  →  inode 1001
```

Two names, **same inode, same data blocks**. This is *not* a copy. It's literally the same file with two labels. (Copy-on-write filesystems like Btrfs/ZFS and dedup features can also share blocks temporarily, but those are special cases.)

### 3.6 A library analogy

| Library | Filesystem |
|---------|-----------|
| Book title on a catalog card | filename (directory entry) |
| The catalog record for the book | inode |
| The actual pages in storage bins | data blocks |

The catalog record (inode) tells you exactly which bins (blocks) hold the pages.

---

## Part 4: Why this matters, the atomic rename trick

This three-layer model explains a pattern you'll see everywhere in robust code (backups, config writers, package managers): **write to a temp file, then rename it over the target.**

```python
copy2(SOURCE, TEMP_FILE)          # 1. write new data to a temp name
os.rename(TEMP_FILE, BACKUP_FILE) # 2. atomically swap the name
```

### 4.1 After the copy

```
data.dat  →  inode 1001   (OLD data)
data.tmp  →  inode 2002   (NEW data)
```

Two separate inodes, two separate sets of data blocks. No confusion.

### 4.2 What `os.rename` actually does

**Rename does NOT move data.** It only changes a directory entry:

```
Before:
  data.dat  →  inode 1001   (OLD)
  data.tmp  →  inode 2002   (NEW)

After  os.rename("data.tmp", "data.dat"):
  data.dat  →  inode 2002   (NEW)
  (data.tmp no longer exists as a name)
```

Key facts:

- inode 2002 still exists. It was never re-copied.
- Only the **name → inode mapping** changed.
- The kernel updates its directory cache immediately, so the OS does **not** "still think `data.dat` is the old data." There is no lingering "old vs new" memory attached to a filename. A name simply points to whatever inode it currently points to.

### 4.3 Why this is "atomic" and crash-safe

`rename()` over an existing name is **atomic** at the filesystem level: at any instant, `data.dat` points to *either* the old inode *or* the new one, never to a half-written file. A reader can never see a corrupted, partially-copied backup.

The remaining subtlety is durability. Consider a power loss at different points:

```
1. copy → TEMP exists (inode 2002 written)
2. rename → BACKUP now points to inode 2002   ← but this may still be only in RAM
3. POWER LOSS before the directory is flushed to disk (fsync)
```

The rename is atomic, but the *result* might still be sitting in the kernel's in-memory cache and not yet on physical disk. After a crash you could find the **old** name mapping survived. That's why durable writers add an explicit flush of the **directory** after the rename:

```python
import os

fd = os.open(SOURCE, os.O_RDONLY)   # or open the file you wrote
os.fsync(fd)                        # flush the file's data blocks
os.close(fd)

os.rename(TEMP_FILE, BACKUP_FILE)   # atomic name swap

dir_fd = os.open(os.path.dirname(BACKUP_FILE), os.O_DIRECTORY)
os.fsync(dir_fd)                    # flush the directory entry itself
os.close(dir_fd)
```

- `fsync(file)` guarantees the **data blocks** hit the disk.
- `fsync(directory)` guarantees the **name → inode mapping** hits the disk.

With both, after a crash you are guaranteed to see *either* a fully-old or a fully-new backup, never a corrupt one. That guarantee falls directly out of the filename → inode → data-blocks model: because the name and the data are separate layers, you can swap the name atomically without ever touching the bytes.

---

## Cheat sheet

```
READING ls -l :  - rw-r--r--  1  alice staff  4096 ...
                 │ └───┬───┘  │  └─┬─┘ └─┬─┘
                 │     │       │    │     └ group owner
                 │     │       │    └ user owner
                 │     │       └ link count (dir = 2 + #subdirs)
                 │     └ 9 permission bits
                 └ type: - file  d dir  l link  c/b device  s socket  p pipe

OWNER / GROUP / OTHER  (the three permission classes)
  owner = the user who owns it          other = everyone else
  group = members of the file's group   you obey the FIRST class you match

  run docker without sudo:  sudo usermod -aG docker $USER   (then re-login)
  -aG = Append to Group. Puts you in group "docker" so you get its rw on
        /var/run/docker.sock. Forgetting -a wipes your other groups!

PERMISSIONS (per class: user / group / other)
  r=4  w=2  x=1   →   add them per digit
  0755 = rwxr-xr-x   scripts, binaries, directories
  0644 = rw-r--r--   normal files
  0664 = rw-rw-r--   group-writable files
  0600 = rw-------   secrets / keys
  leading 0 = no special bits (4=setuid, 2=setgid, 1=sticky)

OWNERSHIP
  -o USER  /  chown USER       sets the owner
  -g GROUP /  chgrp GROUP      sets the group
  -o root -g root  ==  chown root:root   (owner=root, group=root)

WHAT A FILE IS
  filename  →  inode  →  data blocks
  (label)      (perms,     (raw bytes)
                owner,
                block ptrs)

ATOMIC SAVE
  write temp → fsync(file) → rename over target → fsync(dir)
  rename only swaps the name→inode mapping. It never copies data,
  so a reader sees old-or-new, never half-written.
```
