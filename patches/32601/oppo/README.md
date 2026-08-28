# `oppo`

Patches every OPPO-lineage build needs and no other vendor does — OPPO, OnePlus
and realme ship the same kernel hardening, so one set covers them.

**Empty today.** PMG110, the only OPPO target so far, builds stock: ColorOS 16
has nothing like Samsung's KDP, RKP or DEFEX, and nothing in the module had to
change for it beyond what `common` already carries.

The directory exists so that the first patch which does turn up has a home,
rather than being folded into `common` where every other vendor would compile it
too. Until then no build can name `oppo` — a set with no patches in it fails the
build rather than being quietly skipped, so that a misspelled name cannot pass
for a real one.
