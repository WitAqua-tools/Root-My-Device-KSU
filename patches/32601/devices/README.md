# `devices`

One directory per build that needs a delta no other build does, named after the
`id` in the `kernelsu.json` that asks for it — `devices/<id>` — and selected as
`devices/<id>` in that file's `patchSets`.

Keyed by build id rather than by device, because that is what a module is: two
targets which load the same module share one id and one build, so a patch keyed
to only one of them could not be honoured. A delta that genuinely applies to one
of them and not the other means they are different modules and want different
ids.

**Empty today**, and it should stay small. A patch here is one nothing else can
use, so reach for it only after `common` and the vendor sets have been ruled
out. As with any set, a `devices/<id>` with no patches in it fails the build
that names it rather than being quietly skipped.

Empty *here* for a reason worth writing down: the one delta the other two
versions carry -- `quest3`, upstream `953b403a` for `/system/bin/stub_zygote` --
is already in this version's tree. A build that names `devices/quest3` therefore
fails at this pin instead of applying nothing, which is the reminder to drop the
name from its `patchSets` when the pin moves here.

These live here rather than beside the target in the consuming repository for
the same reason the rest do: they are a derivative work of KernelSU and carry
its licence, which is not the licence of the repository that builds them.
