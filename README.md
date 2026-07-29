# Root My Device KSU

Source patches against [KernelSU](https://github.com/tiann/KernelSU), kept in
their own repository so that they carry KernelSU's licence rather than the
Apache-2.0 licence of the projects that consume them.

This repository holds **source only**. It builds nothing; the modules and
`ksud` binaries are built by
[Root-My-Device-Payloads](https://github.com/Witaqua-tools/Root-My-Device-Payloads),
which pins the upstream KernelSU submodule and holds the per-target build
definitions.

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

Three, because that is enough to be able to move: the tag the builds pin, one
step back that still works if a release turns out to regress something, and
`main` ahead of it so the next bump is a pin move rather than a rebase under
time pressure. Adding a fourth means dropping the oldest.

### Moving to a newer upstream

Copy the newest `patches/<old>` to `patches/<new>`, rebase the hunks until they
apply to the new commit, drop the oldest of the four, and move the submodule pin
in Root-My-Device-Payloads.

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
| `devices/<id>/` | the one build that names it | what nothing else can use — [empty today](patches/32525/devices/README.md) |

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
