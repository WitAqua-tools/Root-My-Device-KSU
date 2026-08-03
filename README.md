# Root My Device KSU

Source patches against [KernelSU](https://github.com/tiann/KernelSU), kept in
their own repository so that they carry KernelSU's licence rather than the
Apache-2.0 licence of the projects that consume them.

The kernel modules and the per-target `ksud` binaries are built by
[Root-My-Device-Payloads](https://github.com/Witaqua-tools/Root-My-Device-Payloads),
which pins this repository and upstream KernelSU as submodules and holds the
per-target build definitions. The one thing built **here** is the
[manager](#manager), because what makes it ours is the patches, and they are
here.

## Upstream

Project: <https://github.com/tiann/KernelSU>.

**The upstream version is the directory name.** Every patch lives under
`patches/<version>/`, where `<version>` is KernelSU's own number for the commit
the set was written against — `30000 + git rev-list --count HEAD`, which is the
expression `kernel/Kbuild` compiles into `KSU_VERSION`, the number the module
reports at run time, and the number the manager compares against its
`MINIMAL_SUPPORTED_KERNEL`. Three are kept:

| Version | Upstream commit | Is |
| --- | --- | --- |
| `32567` | `953b403a` | `main` as of 2026-07-29 |
| `32525` | `b0bc817b` | tag `v3.2.5` — **what the builds pin** |
| `32514` | `46ad8dcb` | the oldest that still satisfies the manager |

Each set applies to its own commit exactly, and each is checked with `git apply
--check` before the build that uses it, so a pin moved to a revision nothing has
been rebased onto surfaces as a failure to apply rather than as a silently
different artifact.

### Why those three, and why three

The floor is the manager's, not ours. `Natives.MINIMAL_SUPPORTED_KERNEL` is
`32513` and `requireNewKernel()` also rejects a `uapi_version` that is not the
manager's own; upstream introduced the uapi version at 32513 with a value of 1
and moved it to 2 at 32514. So **32514 is the oldest revision a current manager
will talk to at all**, which makes it the bottom of the window rather than an
arbitrary choice. Anything below it is not a version this repository can carry a
patch for — it would build a module the manager refuses.

Three, because that is enough to be able to move in either direction without a
rebase first: the tag the builds pin, `main` ahead of it so the next bump is a
pin move rather than a rebase under time pressure, and the floor behind it to
fall back to if a release turns out to regress something. Adding a fourth means
dropping the one furthest from the pin.

### Moving to a newer upstream

Copy the nearest `patches/<old>` to `patches/<new>`, rebase the hunks until they
apply to the new commit, drop the fourth, and move the submodule pin in
Root-My-Device-Payloads. Nothing below 32514 may be the one kept, for the reason
above.

The build derives the directory from the pin it already holds, so the version is
never written down twice and cannot disagree with itself — a pin whose directory
is missing fails the build instead of silently taking a set written against
another tree. Reverting is the pin again, and no set has to be deleted before
its replacement is proven.

## Licence

These patches are a derivative work of KernelSU and are distributed under
KernelSU's own terms, which differ by directory. Each hunk is licensed as the
upstream file it modifies, whichever patch file it happens to sit in:

| Hunks under | Licence | Copy in this repository |
| --- | --- | --- |
| `kernel/**` | GNU GPL v2.0 | [`LICENSE.kernel-GPL-2.0`](LICENSE.kernel-GPL-2.0) |
| `userspace/**` | GNU GPL v3.0 | [`LICENSE`](LICENSE) |

Both licence files are verbatim copies of the corresponding files in the
upstream tree at the pinned commit. Nothing here is relicensed, and no
Apache-2.0 terms apply to any file in this repository.

`.github/` is not derived from KernelSU — it is a build definition written here
— and is covered by this repository's [`LICENSE`](LICENSE), the same GPL-3.0 as
the `userspace/**` hunks, so the repository carries one licence for its own
work rather than a third file. What that workflow builds is upstream KernelSU
with the patches applied and carries KernelSU's terms, as it would if it were
built by hand.

## Layout

Patches are keyed first by the upstream version they were written against and
then by who needs them: `patches/<version>/<set>/`. A patch is scoped as
narrowly as it can be, so that a build compiles only what it is actually the
reason for. Each directory under a version is one **patch set**, and a set is
either always applied or named by the build that wants it:

| Set | Applied to | Holds |
| --- | --- | --- |
| `common/` | every build, always, first | what no target can boot without |
| `galaxy/` | builds that name it | Samsung KDP / RKP / DEFEX |
| `oppo/` | builds that name it | OPPO, OnePlus and realme — [empty today](patches/32525/oppo/README.md) |
| `devices/<id>/` | the one build that names it | what nothing else can use — [`quest3` today](patches/32525/devices/README.md) |

Within a set the patches apply in filename order, hence the `NNNN-` prefixes.
The sets apply in the order `common`, then whatever the build named, in the
order it named them.

Nothing outside `common/` is compiled into a build that did not ask for it,
because it is not applied to that build's tree at all. Kernel options such as
`CONFIG_KSU_SAMSUNG_{KDP,RKP,DEFEX}` still gate the code inside a set, but they
gate source that is only present once the set has been applied.

Which sets a build takes is declared where the build is, not here:
Root-My-Device-Payloads reads a `patchSets` array out of each target's
`kernelsu.json`. The names in it are relative to the version directory, so a
build never names a version — it inherits the one its submodule pin selects. A
name that resolves to a directory with no patches in it — because it was
misspelled, or because the set is still a placeholder, or because it was not
carried over in a version bump — fails that build rather than being quietly
skipped.

### `common/`

**Required by every target**, on any vendor's firmware.

`0001-ksud-staged-late-load.patch` — upstream late-load could not write a new
`/data/adb/ksud` after the module changed the loader's security context: the
destination stayed a zero-byte file. The patch stages the daemon at
`/data/local/tmp/.ksud-stage`, renames it onto the same `/data` filesystem
before loading the module, and finishes labels and assets once the module is
active. Removing it breaks installation everywhere.

`0002-kbuild-include-paths.patch` — the include paths a build out of a DDK
image needs, where the SELinux headers are reachable through `objtree` and not
only through `srctree`.

`0003-late-load-module-scripts.patch` — module stage scripts do not run on a
late load unless `KSU_LATE_LOAD_MODULES=1` asks for them. They assume a boot:
their mounts established and no framework yet, and what they do instead is
restart a zygote that is already serving, which on warhol kills `system_server`
until the device is rebooted. KernelSU itself needs none of them. The patch
also rejoins init's mount namespace before touching modules, and logs the one
line naming the version, uapi, flags and features the module reports — evidence
that KernelSU is live which does not depend on descriptors the sepolicy reload
takes away.

`0004-reap-module-daemons-on-soft-reboot.patch` — an emulated soft reboot kills
the daemons a module started before it runs the stage scripts again. Since
`0003` the first cycle a module ever gets is a soft reboot, so every one after
it meets daemons that a real reboot would have taken. Modules are meant to shut
themselves down in the `emulated-soft-reboot` stage — Zygisk Next does — but
most never implement it, and those go one of two ways: LSPosed exits on its
single-instance lock and leaves the stale daemon holding the module directory
that post-fs-data is about to replace, after which it cannot start its bridge
into the new `system_server` at all; Sui starts a second daemon and leaves both.
A process counts as a daemon to reap when its executable, working directory or
any mapping is under `/data/adb/modules/`, and its process group goes with it
unless that group is init's.

### `galaxy/`

`0001-samsung-kdp-rkp-defex.patch` — a generic build panics on Samsung
firmware: an inline `put_cred()` writes directly to a KDP-protected credential
refcount, RKP rejects the write to an unused syscall-table slot while the
generic code still treats the dispatcher as installed, and DEFEX keeps its own
task credential tuple that a KernelSU UID transition leaves unsynchronised.

The patch resolves `kdp_usecount_dec_and_test()` and `kdp_assign_pgd()` from the
running kernel, installs credentials through `prepare_ro_creds()`, synchronises
the DEFEX record, records a syscall-table hook only if the RKP-protected write
succeeds, and otherwise falls back to kretprobe/kprobe sucompat. Each of the
three is behind its own `CONFIG_KSU_SAMSUNG_*` option, which the build passes
alongside the set.

It is written for the Samsung kernels it exists for, and no target in
Root-My-Device-Payloads currently names it. Built against an `android16-6.12`
DDK it does not compile — `compat/samsung_kdp.c` calls `task_pid_nr()` without
the declaration that kernel's headers do not reach transitively. That is true
of all three versions here, including the one none of this restructuring
touched, so it is a property of the set and not of a rebase.

## Manager

The KernelSU manager **rewrites `/data/adb/ksud` with the copy bundled in its
own APK the first time it runs**. On a device Root-My-Device rooted that
silently reverts the patches in `ksud` — measured on Quest 3, where opening the
official manager undid `common/0004` and the next soft restart went back to
leaving stale daemons behind, with nothing in any log to say the daemon had
been replaced. A soft restart is the only restart such a device has, so that is
the normal path rather than a corner case.

The **Manager** workflow builds one whose bundled `ksud` is already the patched
one. It checks out upstream at the revision the patches were written against,
applies `common` to the working tree, builds `ksud` for `aarch64` and `x86_64`,
lets Gradle build the APK and `repack_apk.py` put that `ksud` into it, and
signs the result. Nothing is forked and nothing is committed on top of
upstream, so `git rev-list --count HEAD` is still upstream's and the manager's
`versionCode` matches the module Root-My-Device-Payloads builds.

Only `common` goes in. The sets a target names on top of it are kernel-side, so
a manager built from `common` alone carries the same daemon every target's
build does; a userspace patch landing in a non-common set would break that, and
the workflow is where to notice.

Two things are checked before the APK is worth anything, because both fail
silently on a device:

- **the signing certificate** is the one the modules are built to trust.
  KernelSU picks its manager by hashing the APK's v2 certificate against a size
  and hash compiled into the module (`KSU_EXPECTED_SIZE2` /
  `KSU_EXPECTED_HASH2`, recorded in the target's `kernelsu.json`). An APK signed
  with anything else is not rejected — it is never found, and nothing is logged;
- **the bundled `ksud` really carries the patches**, by a string only
  `common/0004` adds.

A push to `main` touching `patches/*/common/**` or the workflow builds and
checks — but it always builds the pinned version, so a green run after an edit
to some *other* version's `common/` says nothing about that version. To check
one, run the workflow by hand with its revision as `ksu_ref`. Publishing a release is a hand-started run with the box ticked. The
signing key lives in the private `Android-Keys` repository and reaches the
build as repository secrets; it is not in this tree.

## Applying by hand

```sh
git clone https://github.com/tiann/KernelSU
# any of the three commits in the table above; b0bc817b is what the builds pin
git -C KernelSU checkout b0bc817b4e966aa6aa830834eaf6ef765d821d40

# the directory is that checkout's own KernelSU version -- 32525 here
version=$(expr 30000 + "$(git -C KernelSU rev-list --count HEAD)")

# every build
git -C KernelSU apply /path/to/patches/"$version"/common/*.patch
# and then whatever the target asks for, in the order it asks
git -C KernelSU apply /path/to/patches/"$version"/galaxy/*.patch
```

A shallow clone answers `1` there and sends you to `patches/30001`, which does
not exist. That is deliberate: the same count is what the module compiles into
`KSU_VERSION`, so a depth-1 checkout would otherwise produce a module claiming
version 30001 — below the manager's `MINIMAL_SUPPORTED_KERNEL` of 32513, and
rejected on the device rather than in the build.

The build procedure, including the DDK images and the load-time contracts a
module has to satisfy, is documented in Root-My-Device-Payloads under
`src/kernelsu/README.md`.
