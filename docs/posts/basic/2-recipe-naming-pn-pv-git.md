---
title: "How a recipe gets its name and version: PN and PV explained"
date: 2026-07-05
categories:
  - Yocto
  - Basic
---

# How a recipe gets its name and version: PN and PV explained

Every bitbake recipe filename follows a strict pattern, `<PN>_<PV>.bb`, and that's where `PN` and `PV` actually come from. Recipes that track git instead of a numbered release, like `myapp_git.bb`, look like they break the pattern, and this post explains why they don't.

<!-- more -->

---

## Part 0: The one rule, the filename is the recipe's identity

A recipe filename follows a strict pattern:

```
<PN>_<PV>.bb
```

and **bitbake reads the filename to set two variables**:

- **`PN`**: **P**ackage **N**ame, everything **before the last** `_`
- **`PV`**: **P**ackage **V**ersion, everything **after the last** `_` (before `.bb`)

That's it. The filename isn't decoration, it's where PN and PV *come from*.

| Filename | → `PN` | → `PV` (from filename) |
|----------|--------|------------------------|
| `expat_2.5.0.bb` | `expat` | `2.5.0` |
| `foo-tools_1.2.3.bb` | `foo-tools` | `1.2.3` |
| `myapp_git.bb` | `myapp` | `git` |
| `zlib_1.3.1.bb` | `zlib` | `1.3.1` |

Note `foo-tools`: the dash is part of the **name**, because PN is split on the **last** underscore, not the first. If a filename has no underscore at all, PV defaults to `1.0`.

!!! note "Who writes the filename?"
    A human, the recipe **author**. bitbake never generates recipe files, it only *parses* the name you give it. So someone deliberately named the file `myapp_git.bb`, and bitbake then derived `PN=myapp`, `PV=git` from it.

---

## Part 1: `PN`, the package name

`PN` is the identity used everywhere: the build directory, package names, dependency references (`DEPENDS`/`RDEPENDS`), bbappend matching, `bitbake <PN>`. When you run `bitbake myapp`, "myapp" is a PN.

Two recipes can share a PN but differ in PV, that's how multiple versions of the same software coexist (Part 6 covers which one gets picked).

---

## Part 2: `PV`, the package version (and why the filename can be overridden)

`PV` starts from the filename, **but the recipe body can reassign it.** That's the key to why `myapp_git.bb` can still end up versioned as `0.1+git`. Look at what a typical recipe body does:

```
# filename: myapp_git.bb   →  bitbake first sets PV = "git"
PV = "0.1+git"                #  ← then this line OVERRIDES it
SRCREV = "1a2b3c4d5e6f7890abcdef1234567890abcdef12"
```

So there are two stages:

1. **Filename** gives a *default* `PV = "git"`.
2. **Line in the recipe** replaces it with the real `PV = "0.1+git"`.

### Why not just name the file `myapp_0.1+git.bb`?

Two reasons:

1. **The real version often can't fit in a filename.** Git-tracked recipes frequently set `PV = "0.1+git${SRCPV}"`, which expands to include an auto-incrementing counter and a commit hash (something like `0.1+git0+1a2b3c4d5e`). You can't bake a live commit hash into a static filename, so the filename carries a stable placeholder and the body computes the precise version.
2. **`_git` is a deliberate signal.** Seeing `_git` in the filename tells a human at a glance that this recipe tracks a git repo rather than pinning a released tarball version. A clean `_2.5.0` would hide that.

### The `+git` / `${SRCPV}` idiom

When you see a `PV` like `1.0+git` or `1.0+git${SRCPV}`:

- `1.0`: the last upstream version number (or a made-up base).
- `+git`: "plus development commits past that release."
- `${SRCPV}`: a fragment bitbake derives from `SRCREV` (roughly `AUTOINC+<short-sha>`) so each commit produces a distinct version string.

### Related version variables you'll meet

| Variable | Meaning |
|----------|---------|
| `PV` | Package version (from filename, often overridden). |
| `PR` | Package **R**evision, a manual bump (`r0`, `r1`) to force a rebuild/repackage without changing `PV`. Defaults to `r0`, rarely set by hand nowadays. |
| `PE` | Package **E**poch, a tie-breaker set when upstream's versioning scheme *changes* and normal version comparison would otherwise pick the wrong one. Usually empty. |
| `SRCPV` | Version fragment computed from `SRCREV` (see above). |

---

## Part 3: `_git` is a versioning choice, not "the source is git"

This is the part that surprises everyone. **Whether a recipe is named `_git` has nothing to do with whether it fetches over git.** Plenty of `_1.2.3` recipes fetch from git.

The deciding question is: does this recipe pin a specific released version, or does it just track git generically?

- **`foo-tools_1.2.3.bb`** fetches from git (`git://example.com/foo-tools.git` with a pinned `SRCREV`), but that commit **is** release 1.2.3, so the author named it by the release number.
- **`myapp_git.bb`** has no clean upstream release to pin to, it follows a development snapshot, so the author used the `_git` placeholder and set `PV = "0.1+git"` in the body.

| Situation | Filename convention | Example |
|-----------|---------------------|---------|
| Upstream has a release (tarball, **or** a git tag that *is* a release) | `foo_<version>.bb` | `expat_2.5.0.bb`, `foo-tools_1.2.3.bb` |
| Recipe just tracks git, no formal release, meant to float | `foo_git.bb` | `myapp_git.bb` |
| Other VCS, same idea | `foo_svn.bb`, `foo_hg.bb` | |

So `_git` means "versioned as a git-tracking recipe." The *fetch method* is set separately by `SRC_URI`/`SRCREV`, a `foo_1.2.3.bb` and a `foo_git.bb` can both use the git fetcher.

---

## Part 4: The `PN`/`PV` family, `BPN`, `BP`, `P`

Several derived variables are built from `PN` and `PV`. You'll see them in `SRC_URI`, `S`, and `FILES`:

| Variable | Definition | Example (`foo_1.0.bb`) | Where it shows up |
|----------|-----------|------------------------|-------------------|
| `PN` | package name (may carry a class prefix/suffix) | `foo` | `bitbake ${PN}`, `FILES:${PN}` |
| `PV` | package version | `1.0` | tarball URLs, tags |
| `BPN` | **B**ase PN, `PN` with class affixes stripped (`-native`, `nativesdk-`, `-cross`, multilib `lib32-`, …) | `foo` | repo/tarball names: `git://…/${BPN}` |
| `BP` | **B**ase **P**refix = `${BPN}-${PV}` | `foo-1.0` | default source dir: `S = ${WORKDIR}/${BP}` |
| `P` | `${PN}-${PV}` | `foo-1.0` | older naming |

**Why `BPN` vs `PN`?** When a recipe is reused to build a *native* tool or an SDK variant, bitbake sets `PN` to `foo-native` / `nativesdk-foo`. But the **source tarball is still called `foo-1.0.tar.gz`**, so URLs use `BPN` (`foo`), which strips the `-native`/`nativesdk-` affix. `BPN` is "the name upstream actually uses."

---

## Part 5: Why the bbappend is `myapp_%.bbappend` (matching by filename)

Filename identity and version overrides connect here. A `.bbappend` attaches to a base `.bb` **by matching PN and PV, and it matches against the *filename*, not the body-overridden `PV`.**

The append filename is `<PN>_<PV>.bbappend`, where `%` is a wildcard for the version:

| bbappend filename | Matches | Because |
|-------------------|---------|---------|
| `myapp_git.bbappend` | `myapp_git.bb` | filename PV is literally `git` ✓ |
| `myapp_0.1+git.bbappend` | *nothing* | the filename version is `git`, not `0.1+git` ✗ |
| `myapp_%.bbappend` | `myapp_git.bb` (and any future `myapp_*`) | `%` = any version ✓ |

This is why you can't name the append after the "real" version `0.1+git`, matching uses the **filename** version (`git`), which is exactly why the recipe keeps a stable placeholder in its name (Part 2). And it's why `%` is so common in bbappends: it survives an upstream version bump without a rename.

!!! note
    Both `_git` and `%` live in the **version slot** of the filename. `_git` matches one specific version, `%` matches all versions. That's the whole relationship between filename identity and bbappend matching.

---

## Part 6: When several recipes provide the same thing (which one wins)

Because PN/PV come from filenames, a layer set can contain **multiple recipes for the same software** (e.g. `foo_1.0.bb` and `foo_2.0.bb`, or a stable `foo_1.2.3.bb` alongside a `foo_git.bb`). bitbake picks one:

| Variable | Meaning |
|----------|---------|
| *(default)* | Highest `PV` wins (version-sorted). |
| `PREFERRED_VERSION_foo = "1.2.3"` | Force a specific version of `foo` (supports `%`, e.g. `"1.2.%"`). |
| `DEFAULT_PREFERENCE = "-1"` | Set **inside** a recipe to say "don't pick me by default." Common in `_git` recipes so the stable release is chosen unless you *explicitly* `PREFERRED_VERSION` the git one. |
| `PROVIDES` | Extra names a recipe advertises (a recipe can provide `virtual/kernel`, etc.). |
| `PREFERRED_PROVIDER_virtual/foo = "myfoo"` | Pick which recipe satisfies a `virtual/…` or otherwise multiply-provided name. |

So a `_git` recipe usually carries `DEFAULT_PREFERENCE = "-1"`: it exists for people who want bleeding-edge, but the versioned release wins unless you ask for git.

---

## Part 7: See the resolved identity yourself

```bash
# what PN/PV/BPN did this recipe actually resolve to?
$ bitbake-getvar -r myapp PN
$ bitbake-getvar -r myapp PV        # → 0.1+git (the body override, not "git")
$ bitbake-getvar -r myapp BPN

# which versions of a recipe exist across all layers, and which is preferred?
$ bitbake-layers show-recipes myapp

# find the recipe file(s) on disk (works even when the config won't parse):
$ find sources -path '*/myapp/*' -name 'myapp*.bb*'
```

`bitbake-layers show-recipes <pn>` is the quickest "how many versions are there and who wins" check. `find` is the fallback while the config still won't parse.

---

## Cheat sheet

| Question | Answer |
|----------|--------|
| Where do `PN`/`PV` come from? | The **filename** `<PN>_<PV>.bb` (split on the **last** `_`). Author-named, bitbake parses it. |
| Can `PV` differ from the filename? | Yes, a line in the recipe body overrides it (`myapp_git.bb` → `PV = "0.1+git"`). |
| What is `_git`? | The filename's PV slot set to the placeholder `git`, meaning "tracks git, no pinned release." |
| Why not `myapp_0.1+git.bb`? | Real PV can include a live commit hash (can't go in a filename), and `_git` is a readable signal. |
| Other git recipes not named `_git`? | They pin a *released* version, so they're named by it (`foo-tools_1.2.3.bb`). `_git` does not mean "source is git." |
| What matches `_%.bbappend`? | Any version, matching is on the **filename** PV. `%` wildcards it, `_git` matches only `_git.bb`. |
| Which recipe wins if several exist? | Highest `PV`, unless `PREFERRED_VERSION`/`DEFAULT_PREFERENCE`/`PREFERRED_PROVIDER` say otherwise. |
