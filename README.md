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
upstream file it modifies:

| Hunks under | Licence | Copy in this repository |
| --- | --- | --- |
| `kernel/**` | GNU GPL v2.0 | [`LICENSE.kernel-GPL-2.0`](LICENSE.kernel-GPL-2.0) |
| `userspace/**` | GNU GPL v3.0 | [`LICENSE`](LICENSE) |

Both licence files are verbatim copies of the corresponding files in the
upstream tree at the commit above. Nothing here is relicensed, and no
Apache-2.0 terms apply to any file in this repository.

## Contents

### `patches/KernelSU-v3.2.5-samsung-kdp-rkp-defex.patch`

One patch carrying two independent halves.

The **`userspace/ksud` half is required by every target**, on any vendor's
firmware. Upstream late-load could not write a new `/data/adb/ksud` after the
module changed the loader's security context — the destination stayed a
zero-byte file. The patch stages the daemon at `/data/local/tmp/.ksud-stage`,
renames it onto the same `/data` filesystem before loading the module, and
finishes labels and assets once the module is active.

The **`kernel/` half is compiled out unless a target enables it**, through
`CONFIG_KSU_SAMSUNG_{KDP,RKP,DEFEX}`. It exists because a generic build panics
on Samsung firmware: an inline `put_cred()` writes directly to a KDP-protected
credential refcount, RKP rejects the write to an unused syscall-table slot while
the generic code still treats the dispatcher as installed, and DEFEX keeps its
own task credential tuple that a KernelSU UID transition leaves unsynchronised.
Under those options the patch resolves `kdp_usecount_dec_and_test()` and
`kdp_assign_pgd()` from the running kernel, installs credentials through
`prepare_ro_creds()`, synchronises the DEFEX record, records a syscall-table
hook only if the RKP-protected write succeeds, and otherwise falls back to
kretprobe/kprobe sucompat.

## Applying by hand

```sh
git clone https://github.com/tiann/KernelSU
git -C KernelSU checkout b0bc817b4e966aa6aa830834eaf6ef765d821d40
git -C KernelSU apply /path/to/patches/KernelSU-v3.2.5-samsung-kdp-rkp-defex.patch
```

The build procedure, including the DDK images and the load-time contracts a
module has to satisfy, is documented in Root-My-Device-Payloads under
`src/kernelsu/README.md`.
