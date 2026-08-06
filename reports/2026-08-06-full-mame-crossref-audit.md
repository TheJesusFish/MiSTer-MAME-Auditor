# Full audit: every mister-devel `.mra` vs. current MAME source — 2026-08-06

Unlike the [changelog-driven 0.289 audit](2026-08-06-mame-0289-audit.md), this
pass doesn't rely on reading release notes at all. It cross-references **every
setname in mister-devel and `MRA-Alternatives_MiSTer` against MAME's current
source directly** — catching drift accumulated over any number of past
releases, not just the latest one. Updated same day with a second pass: git
history archaeology on everything that came out ambiguous the first time.

## Method (folded into [mame.md](../mame.md))

GitHub's `api.github.com` rate limit (60/hour unauthenticated) makes it
impractical to inspect ~150+ individual driver files one fetch at a time. The
fix: MAME's source is a normal git repo, so pull it the same
rate-limit-free way as the MiSTer repos:

```bash
git clone --filter=blob:none --no-checkout --depth 1 -q https://github.com/mamedev/mame.git mame_src
cd mame_src
git sparse-checkout init --no-cone
git sparse-checkout set 'src/mame/**/*.cpp'
git checkout
```

This pulls all ~4,660 driver `.cpp` files (~150 MB, ~8 seconds) with no binary
assets. From there, every `GAME(`/`GAMEL(` macro invocation in the entire tree
was regex-extracted into `(setname, parent, year, source_file)` — 31,782 rows,
covering the entirety of current MAME, not just arcade. Saved as
[`data/mame-current-sets.tsv`](../data/mame-current-sets.tsv) — a dated
snapshot of MAME truth, same staleness caveat as the MiSTer-side snapshots.

**Watch out if you reimplement this regex:** MAME setnames can start with a
digit (`99lstwar`, `4dwarrio`, `280zzzap` are all real) — don't anchor the
first character to `[A-Za-z_]` or you'll silently lose a percent or so of
sets and generate false "missing" findings. (Caught and fixed during this
run — the first pass wrongly flagged all three of those as gone.)

Every mister-devel/`MRA-Alternatives` setname was then checked for existence
in this table, and where found, its declared `<parent>` was compared.

## Two things this method assumes that turned out to be wrong

### 1. `<parent>` mismatches are not a usable signal on their own

886 of 1,848 entries "mismatched" on parent by naive comparison — but manual
inspection showed this is overwhelmingly a MiSTer authoring convention, not
drift. MiSTer `.mra` files bundle complete ROM data directly (unlike MAME's
merged/split zip inheritance), so the `<parent>` tag is optional documentation,
filled in inconsistently: some root/parent releases self-reference their own
setname in `<parent>` instead of leaving it blank; many real MAME clones ship
with no `<parent>` tag at all. **Don't treat a parent-tag mismatch as a finding
by itself** — it's only meaningful together with a setname that's actually
missing.

### 2. Most of `MRA-Alternatives_MiSTer`'s "missing" setnames are HBMAME, not MAME

207 of 223 "missing" setnames in `MRA-Alternatives_MiSTer` turned out to be
[HBMAME](https://hbmame-roms.com/) content — a separate hack/homebrew/trainer
fork of MAME, not mainline MAME. These were never going to match
`mamedev/mame` source and this isn't drift. **The tell is reliable and
mechanical: the `.mra` filename itself contains `HBMame`** (e.g. `Donkey Kong
Foundry - HBMame.mra`). Filter these out by filename before treating anything
as a finding.

## What's out of scope, on purpose

A "missing setname" isn't automatically a finding. Two categories get checked
once and then dropped, not carried forward as open items:

- **HBMAME and other hack/patch content** — filename says `HBMame`, or the
  description says `Trainer`/`Free Play`/`(inv)`/`ROM Patch`/similar. Never an
  official MAME romset; nothing to sync.
- **Content MAME doesn't emulate at all** — e.g. `spacerace` (Arcade-SpaceRace_MiSTer)
  is a 1973 Atari discrete-logic TTL game with no ROMs; MAME's
  `atarittl.cpp` documents it but has no working machine for it. MiSTer's
  implementation doesn't depend on MAME here, so there's nothing to check
  against.

This run's instances of each, checked and closed, not tracked further:
`rtype2inv`, `xmultiplm72inv`, `cleansweept`, `mrdonight`, `tutankhm2`,
`spacerace` (core repos); `athenaff`, `ddonpachjt`, `esprade_fp`,
`espradej_fp`, and the 207 HBMAME-filename alternatives (`MRA-Alternatives_MiSTer`).
If a future pass turns up more of these, same treatment — verify once, drop.

## Rename archaeology: chasing the ambiguous cases through MAME's git history

The first pass left several setnames that don't exist in *current* MAME with
no obvious replacement — same shape as the TwinBee case (which we already
knew the answer to), but without a changelog line pointing at the answer. For
those, the question is exactly what the TwinBee case answers by hand: **was
this ever a real MAME setname that got renamed at some point, and did we —
or whoever maintains the `.mra` — just never catch it?**

`git log -S` (pickaxe: find the commit that added/removed a given string) on
a local clone answers this directly, but a normal clone of MAME's full history
is impractically large. The fix is the same shape as the rate-limit workaround
above — a partial clone, just filtering differently:

```bash
git clone --filter=blob:none --no-checkout https://github.com/mamedev/mame.git mame_hist
```

No `--depth` this time (we need full history), `--filter=blob:none` instead of
`tree:0` (trees stay local so `git log` traversal is fast; only blob content
is fetched on demand, which is only needed for `-S`/`-p`). This pulled the
**entire commit graph and tree history for all of MAME** — every commit,
every tree — in about **210 MB and 9 seconds**. (A `tree:0` filter, which
defers trees too, was tried first and was unusably slow — every `git log`
step became a network round-trip. Trees need to be local; only blobs can stay
lazy.)

For each ambiguous setname, against its driver file:

```bash
git log -S"<setname>" --follow --oneline -- src/mame/<mfg>/<driver>.cpp
```

`--follow` matters — MAME went through a large driver reorganization in 2022
that moved most files from `src/mame/drivers/*.cpp` into manufacturer
subfolders (e.g. `src/mame/drivers/bagman.cpp` → `src/mame/valadon/bagman.cpp`),
and several of these renames predate that move. `--follow` walks straight
through it. Each search took roughly 1–2 minutes; a couple, on the very large
`williams.cpp`, needed backgrounding rather than blocking. This is not
something to run for all ~1,850 setnames — it's for the handful that survive
the cheap existence check above and still need an answer.

**Result: 7 of the 10 ambiguous setnames were real MAME sets, renamed at a
specific point in MAME's history, unrelated to 0.289 and unrelated to each
other. All confirmed by reading the actual diff, not just guessed from the
commit message:**

| Old setname | New setname | Renamed | Commit | Evidence |
|---|---|---|---|---|
| `kengoa` | `kengoj` | 2023-03-19 | `f5f7689a20` | Diff shows `ROM_START( kengoa )`→`ROM_START( kengoj )` directly; description changed from "Ken-Go (set 2)" to "Ken-Go (Japan)". |
| `pengo2` | `pengoa` | 2022-12-29 | `074670cd81` | Comment text ("Uses Sega 315-5010 encrypted Z80 CPU") carried over unchanged between old and new entries. |
| `pengo4` | `pengoc` | 2022-12-29 | `074670cd81` | Same commit; comment text ("Sega game ID# 834-5081... REV.A of this set known to exist, but not currently dumped") matches verbatim old→new. |
| `pengo5` | `pengob` | 2022-12-29 | `074670cd81` | Same commit; comment text ("Sega game ID# 834-5081... Bally N.E.") matches verbatim old→new. |
| `bagmans2` | `bagmans4` | 2020-10-04 | `1f2cf06ae5` | `ROM_START( bagmans2 )`→`ROM_START( bagmans4 )` directly in the diff; ROM filename `a4_9e.bin` confirms it's the "A4" revision. |
| `joustwr` | `jousty` | 2020-08-28 | `4e0c82dad8` | Not just a rename — a re-identification. Old: "Joust (White/Red label)". New: "Joust (Yellow label)". MiSTer's file is still named/set for the old, now-incorrect identification. |
| `sinistar1` | `sinistarp` | 2020-08-05 | `4ea74b81a3` | Old: "Sinistar (prototype version)". New: "Sinistar (AMOA-82 prototype)" — same set, corrected/expanded description. |

(Commit dates/hashes are from `mamedev/mame`'s `master` branch history.)

**3 of the 10 could not be found anywhere in the driver's full tracked
history** (all the way back to MAME's initial git import, ~2008, confirmed by
checking each file goes back that far): `spclone`/`spcloneo`
(`src/mame/konami/nemesis.cpp`, Salamander family) and `ironhorsbl`
(`src/mame/konami/ironhors.cpp`). A repo-wide (`--all`, no path) search for
`spclone` was attempted too, to rule out it having lived under some unrelated
filename, but timed out — that search space is too large for this technique
and wasn't pursued further. Best current read: these were never official MAME
setnames — likely MiSTer-community-only labels for content that was hand-built
or hacked before/without MAME support — but that's inference, not confirmed
the way the seven above are. Flagged as such, not chased further.

## Priority summary (supersedes the first pass — see mame.md's "Open findings" for the live version)

**Confirmed renames, exact fix known:**

`twinbeeb`→`bs_twinbee` · `devilfsg`→`devilfshg` · `gauntletr8`→`gauntletgr8`
(both Arcade-Gauntlet_MiSTer and `MRA-Alternatives_MiSTer`) · `imgfightb`→`imgfightjb` ·
`popflamn`→`popflamen` · `SpaceDemon`→`spacedem` · `kengoa`→`kengoj` ·
`pengo2`→`pengoa` · `pengo4`→`pengoc` · `pengo5`→`pengob` · `bagmans2`→`bagmans4` ·
`joustwr`→`jousty` · `sinistar1`→`sinistarp`.

**Could not confirm despite a full git-history search — likely never an
official MAME setname, not pursued further:** `spclone`, `spcloneo`,
`ironhorsbl`.

## CRC verification: which of the 13 renames also need ROM content changes

A setname rename in MAME doesn't automatically mean the bytes changed —
sometimes it's purely a relabel. Checked by diffing each affected `.mra`'s
declared CRCs against current MAME's `ROM_START` for the new setname:

- **Pure rename, CRCs identical, no content change needed:** `SpaceDemon`→`spacedem`,
  `pengo2`→`pengoa`, `pengo4`→`pengoc`, `pengo5`→`pengob`, `joustwr`→`jousty`,
  and `devilfsg` — which turned out to need correcting to **`devilfshgb`**
  (the bootleg clone), not `devilfshg` (the Galaxian-hardware clone) as
  first guessed from the name alone; CRCs match `devilfshgb` exactly (7/7),
  match `devilfshg` on none (0/13). `gauntletr8`→`gauntletgr8` is also a pure
  rename as far as the existing `.mra` content goes (all 17 old CRCs present
  in the new set); MAME's current definition adds 3 more small PROM files
  (timing lookup tables) beyond what either existing `.mra` copy has, not
  pursued as part of the rename fix.
- **Rename + real content change, needs a rebuild:** `twinbeeb`→`bs_twinbee`
  (excluded from this batch — see below), `imgfightb`→`imgfightjb` (24 of 25
  ROMs identical; one program chip, `ic111.9e`, has a corrected dump:
  `da50622e`→`6aae3a46`), `bagmans2`→`bagmans4` (rename plus one color PROM
  redump at the same position, `2a855523`→`47504204`; current MAME also
  defines a new `5110ctrl` region — a TMS5110 state-machine PROM — not
  present in the `.mra` at all).
- **Rename + additional ROM region MAME now includes, not present in the
  `.mra` at all:** `kengoa`→`kengoj` (+6 CRCs), `sinistar1`→`sinistarp` (+4
  CRCs, a "soundcpu" region MAME defines as 5 separate speech chips where the
  `.mra` has 1), `popflamn`→`popflamen` (+2 CRCs).

## Applied — 2026-08-06

Scope for this batch, per instruction: **only `MRA-Alternatives_MiSTer`
clone `.mra`s, plus genuine flagship-clone `.mra`s sitting directly in a main
repo's `releases/`** (not bonus/alternate games bundled in a main repo, and
not `_alternatives`-folder copies vendored inside a main repo). `twinbeeb`
excluded — a PR already exists upstream for it.

**Fixed, committed locally (not yet pushed/PR'd upstream):**
- `MRA-Alternatives_MiSTer` (commit `9ac7a0a`): `pengo2`→`pengoa`,
  `pengo4`→`pengoc`, `pengo5`→`pengob` (setname/parent/mameversion only,
  files renamed `set 2/4/5`→`set 1/2/3` to match MAME's current
  descriptions); `joustwr`→`jousty` (setname/parent/mameversion only, file
  renamed to match the corrected label); `bagmans2`→`bagmans4`
  (setname/parent/mameversion plus the one corrected PROM CRC;
  `5110ctrl` region gap documented in `<about>`, not guessed at);
  `sinistar1`→`sinistarp` (setname/parent/mameversion; the 4-chip
  `soundcpu` gap documented in `<about>`, not guessed at); deleted the
  stale duplicate `Gauntlet (DE, Rev 8).mra` (`gauntletr8`), keeping the
  already-correct `Gauntlet (German, rev 8).mra` (`gauntletgr8`) that
  existed alongside it.
- `Arcade-SpaceFirebird_MiSTer` (commit `31f25fa`): `SpaceDemon`→`spacedem`,
  `SpaceFirebird`→`spacefb` (setname/parent tags only — the `<rom zip=...>`
  attribute already used the correct lowercase names, only the identifying
  tags were wrong).

Deliberately not fixed in this batch: `twinbeeb` (upstream PR exists),
`devilfsg`→`devilfshgb`, `popflamn`→`popflamen`, `imgfightb`→`imgfightjb`,
`kengoa`→`kengoj`, and Arcade-Gauntlet_MiSTer's own internal
`_alternatives`-mirror copy of `gauntletr8` — all out of scope per the
working rule above, not overlooked. See mame.md's "Open findings" for the
live status of all of these.

Not done as part of this pass: forking the two repos under a GitHub account,
pushing these commits, and opening PRs against MiSTer-devel. The `.mra`
edits exist only in local clones so far.

**Not tracked (checked once, closed — see "What's out of scope" above):**
`rtype2inv`, `xmultiplm72inv`, `cleansweept`, `mrdonight`, `tutankhm2`,
`spacerace`, `athenaff`, `ddonpachjt`, `esprade_fp`, `espradej_fp`, and all
207 HBMAME-filename `MRA-Alternatives_MiSTer` entries.

## What wasn't checked

- **CRC/content-level verification** was only done for the entries above plus
  the two cases already confirmed in the 0.289 report. The ~1,600 of 1,848
  entries that matched on setname weren't individually re-verified against
  source CRCs — that's a much larger parsing effort (MAME `ROM_LOAD` macros
  are frequently shared via `#define` blocks across multiple sets, easy to
  mis-parse) and wasn't judged worth it without a specific reason to suspect
  a given set.
- Jotego-framework cores are still out of scope (see mister-devel.md).
