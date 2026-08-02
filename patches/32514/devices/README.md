# `devices`

One directory per build that needs a delta no other build does, named after the
`id` in the `kernelsu.json` that asks for it — `devices/<id>` — and selected as
`devices/<id>` in that file's `patchSets`.

Keyed by build id rather than by device, because that is what a module is: two
targets which load the same module share one id and one build, so a patch keyed
to only one of them could not be honoured. A delta that genuinely applies to one
of them and not the other means they are different modules and want different
ids.

**One directory today**, `quest3`, and it should stay small. A patch here is one
nothing else can use, so reach for it only after `common` and the vendor sets
have been ruled out. As with any set, a `devices/<id>` with no patches in it
fails the build that names it rather than being quietly skipped.

`quest3` is upstream `953b403a` backported onto this version: it keeps the
tracepoint mark on `/system/bin/stub_zygote`, which is how Meta's Horizon OS
starts the zygote. Unmarking the stub unmarks the zygote and every application
forked from it, so no KernelSU hook fires for any of them. That survives a boot
-- the module marks the running processes as it loads -- and does not survive
`ksud soft-reboot`.

These live here rather than beside the target in the consuming repository for
the same reason the rest do: they are a derivative work of KernelSU and carry
its licence, which is not the licence of the repository that builds them.
