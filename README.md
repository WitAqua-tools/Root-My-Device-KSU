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

| | |
| --- | --- |
| Project | <https://github.com/tiann/KernelSU> |
| Tag | `v3.2.5` |
| Commit | `b0bc817b4e966aa6aa830834eaf6ef765d821d40` |

The patches apply to that exact commit. They are checked with `git apply
--check` before every build, so a change of upstream revision surfaces as a
failure to apply rather than as a silently different artifact.

## Licence

These patches are a derivative work of KernelSU and are distributed under
KernelSU's own terms, which differ by directory. Each hunk is licensed as the
upstream file it modifies, whichever patch file it happens to sit in:

| Hunks under | Licence | Copy in this repository |
| --- | --- | --- |
| `kernel/**` | GNU GPL v2.0 | [`LICENSE.kernel-GPL-2.0`](LICENSE.kernel-GPL-2.0) |
| `userspace/**` | GNU GPL v3.0 | [`LICENSE`](LICENSE) |

Both licence files are verbatim copies of the corresponding files in the
upstream tree at the commit above. Nothing here is relicensed, and no
Apache-2.0 terms apply to any file in this repository.

## Layout

A patch is scoped as narrowly as it can be, so that a build compiles only what
it is actually the reason for. Each directory under `patches/` is one **patch
set**, and a set is either always applied or named by the build that wants it:

| Set | Applied to | Holds |
| --- | --- | --- |
| `common/` | every build, always, first | what no target can boot without |
| `galaxy/` | builds that name it | Samsung KDP / RKP / DEFEX |
| `oppo/` | builds that name it | OPPO, OnePlus and realme — [empty today](patches/oppo/README.md) |
| `devices/<id>/` | the one build that names it | what nothing else can use — [empty today](patches/devices/README.md) |

Within a set the patches apply in filename order, hence the `NNNN-` prefixes.
The sets apply in the order `common`, then whatever the build named, in the
order it named them.

Nothing outside `common/` is compiled into a build that did not ask for it,
because it is not applied to that build's tree at all. Kernel options such as
`CONFIG_KSU_SAMSUNG_{KDP,RKP,DEFEX}` still gate the code inside a set, but they
gate source that is only present once the set has been applied.

Which sets a build takes is declared where the build is, not here:
Root-My-Device-Payloads reads a `patchSets` array out of each target's
`kernelsu.json`. A name that resolves to a directory with no patches in it —
because it was misspelled, or because the set is still a placeholder — fails
that build rather than being quietly skipped.

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

## Applying by hand

```sh
git clone https://github.com/tiann/KernelSU
git -C KernelSU checkout b0bc817b4e966aa6aa830834eaf6ef765d821d40

# every build
git -C KernelSU apply /path/to/patches/common/*.patch
# and then whatever the target asks for, in the order it asks
git -C KernelSU apply /path/to/patches/galaxy/*.patch
```

The build procedure, including the DDK images and the load-time contracts a
module has to satisfy, is documented in Root-My-Device-Payloads under
`src/kernelsu/README.md`.
