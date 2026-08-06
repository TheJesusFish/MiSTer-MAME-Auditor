# `zip=` completeness audit: MRA-Alternatives_MiSTer — 2026-08-06

Separate from renames: checked whether every `.mra`'s primary
`<rom index="0">` `zip=` attribute actually lists both its parent's
(merged-style) zip and its own (non-merged) zip. This isn't a MAME-change
question at all — it's a standing correctness check on `MRA-Alternatives_MiSTer`
itself, prompted by a direct question about it. Full method now lives in
mame.md's "Checking `zip=` completeness" section; this is the record of the
one pass that established it.

## Two false-positive traps found before trusting any results

1. **Comparing against the `.mra`'s own `<parent>` XML tag produces a wall of
   false positives.** That tag is often a MiSTer-chosen display/grouping
   label, not MAME's actual romset parent — e.g. `zigzagb2` declares
   `<parent>digdug</parent>` (UI grouping: Zig Zag runs on Dig Dug-derived
   hardware) but MAME's real parent is `zigzagb`, and the `zip=` reference
   correctly uses `zigzagb.zip`. An initial pass using the `<parent>` tag
   flagged 44 files as "missing their parent's zip"; checking against
   `data/mame-current-sets.tsv` instead (MAME's actual current parent) cut
   that to 1 real case, plus one more that turned out to be a multi-level
   BIOS-chain false positive (see below).
2. **Attribute quoting isn't consistent across the repo.** Most files use
   `zip="..."`, some use `zip='...'`. A regex anchored to double quotes
   silently undercounts — missed 28 files' zip attributes entirely on the
   first pass, several of them (`spclone`, `ironhorsbl`) among the already-known
   unresolved cases, which made them look like a structural problem when
   they weren't.

## The order convention, confirmed rather than assumed

Some `.mra`s self-document their `zip=` structure with a `type=` attribute
on the same tag — e.g. `type="merged|nonmerged"` — which literally labels
what each pipe-separated position represents, in order. All 31 files that
declare this explicitly use **parent(merged) first, clone's own(non-merged)
second**, except one outlier. It's also the majority pattern (~3:1) among
the ~500 files that don't declare `type=` at all. Confirmed this before
touching anything, since the alternative order was raised as a possibility
and turned out to be backwards relative to the repo's own established
convention.

## What was fixed

40 files were missing the clone's own (non-merged) zip reference — only the
parent's zip was listed, e.g. `zip="boogwing.zip"` with no `boogwinga.zip`.
Inserted the missing name at position 1 (right after the parent's entry),
purely additive — any other existing entries (subboard BIOS zips, an
alternate byte-identical source like `arkatayt.zip` on the Arkanoid clones)
were left exactly as they were, since understanding *why* they're there
wasn't necessary to know the clone's own name was also legitimately missing.

1 file (`Super Locomotive (old)`, setname `suprlocoo`) was missing the
parent's zip (`suprloco.zip`) — prepended it.

1 file (`Super Real Mahjong VS`, setname `srmvsa`) referenced its BIOS root
directly (`aleck64.zip`), skipping two intermediate levels (`srmvs`, its own
`srmvsa`) — added both anyway, additively, though this one wasn't strictly
broken; the existing single reference already worked for merged-style
downloads. Confirmed `srmvsa.zip` is a real, legitimate name independently —
it's already referenced on a different `<rom index>` in the same file.

## What wasn't touched

- ~200 setnames referenced in the repo that don't exist in current MAME at
  all (HBMAME/hack content, or the small number of already-tracked
  unresolved cases like `spclone`/`spcloneo`/`ironhorsbl`) — out of scope,
  consistent with existing policy.
- 8 multi-region `.mra` files (Decathlete, Die Hard Arcade, Tecmo World Cup
  '98, and similar ST-V-family games) whose actual ROM content lives on
  `<rom index="1">`/`<rom index="2">`, not index 0 — this pass only checked
  the primary index-0 tag, consistent with how every other `.mra` in the
  repo is structured, but these use a different layout entirely and weren't
  evaluated.
- Any `type=` attribute that's now technically incomplete (e.g. still says
  `type="merged"` where a second zip entry was added) — left alone; not
  confident enough about whether MiSTer's build tooling actually consumes
  that field to edit it without understanding the consequences.

Committed and pushed to the `MRA-Alternatives_MiSTer` fork, on top of the
earlier rename fixes from the same day.
