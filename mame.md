# mame.md — the MAME side of the audit

How to find out what changed in a MAME release, and where this audit last left
off. Read [prompt.md](prompt.md) for the overall task; read
[mister-devel.md](mister-devel.md) for the MiSTer side.

## Finding the current version

`https://www.mamedev.org/release.html` names the latest official release
(e.g. "MAME 0.289"). MAME ships roughly monthly but sometimes skips a month —
don't assume the version number one run ago plus one is necessarily current.

## The changelog

Each release has a full changelog at:

```
https://www.mamedev.org/releases/whatsnew_0XXX.txt
```

(zero-padded version, e.g. `whatsnew_0289.txt`). This is plain text, one file
per version — there is no cumulative diff endpoint, so catching up N versions
means reading N of these.

**Important caveat, confirmed by reading 0.289's in full:** there is no
structured "renamed machines" or "reparented clones" section anymore (older
MAME versions had one; it's gone now). Renames, reparenting, and romset splits
are described in prose, scattered across `MAME Testers bugs fixed`, `Systems/
Clones promoted to working`, and — mostly — the free-form `Source changes`
section that makes up the bulk of the file. There is no way to mechanically
diff this; it has to be read. Budget for that: 0.289's file was ~4,600 lines /
265 KB, which is small enough to read in full directly rather than sample.

### What to scan for

A full read is better than keyword grepping alone (grepping is what surfaced
the leads below, but always sanity-check by reading the surrounding section —
short commit-message-style lines compress a lot). Keywords worth searching, in
rough order of how often they've mattered so far: `renam`, `reparent`,
`made <x> the parent`, `clone of`, `split`, `merged into`, `romset`,
`corrected (the )?(manufacturer|year|title)`, `documentation`. Also always read
`New working clones`, `Clones promoted to working`, and `New clones marked not
working` in full — these are short lists and are where brand-new romset
variants worth adding to `MRA-Alternatives_MiSTer` tend to show up (opportunity,
not breakage).

Distinguish, for anything you find, between:

- **Romset-breaking** — a setname changed, a clone got reparented, ROM content
  moved between zips. These need an `.mra` update.
- **Cosmetic** — a display title/manufacturer/year correction, an internal
  C++ state-class rename, a MAME *source file* rename/split with no setname
  change (this happens almost every release and is usually a no-op for
  MiSTer — see mister-devel.md's note on `<rbf>` vs. driver file paths).
  Don't report these as breakage; a one-line mention is enough if at all.

### Verifying a lead against current MAME source

Don't trust the prose description alone for exact old/new setnames — confirm
against the driver source, which is the ground truth and doesn't need a MAME
binary to read:

```
https://raw.githubusercontent.com/mamedev/mame/master/src/mame/<mfg>/<driver>.cpp
```

Grep it for `^GAME(` and `^ROM_START(` — the `GAME()` macro's 2nd and 3rd
positional args are `(setname, parent, ...)`. `parent` of `0` means it's a
root/parent set itself. A driver's manufacturer subfolder (`konami/`, `atlus/`,
`dataeast/`, etc.) is usually named in the changelog line itself
(`atlus/cave.cpp: ...`); if it isn't, the MAME source tree at
`https://github.com/mamedev/mame/tree/master/src/mame` is browsable directly.

This is also how you catch cases where the changelog undersells the change —
e.g. 0.289's "Made the Japanese Sailor Moon game the parent" turned out to mean
the ROM *content* under the `sailormn` setname moved (was EU, is now Japan),
with the old EU content relocated to a new `sailormne` setname — not just a
parent pointer flip. That distinction is the difference between "update a tag"
and "every alt `.mra` needs its zip contents re-checked."

## Full cross-reference against current MAME (not just the changelog)

The changelog method above catches what changed *in one release*. It won't
catch drift that's been sitting there since some earlier release nobody
audited. For that, cross-reference every setname mister-devel/`MRA-Alternatives`
references against MAME's *current* state directly — this doesn't require
reading any changelog at all, and it's cheap enough to do in full, not sampled.

Pull the whole current MAME driver source locally, the same rate-limit-free
way as the MiSTer repos (see mister-devel.md):

```bash
git clone --filter=blob:none --no-checkout --depth 1 -q https://github.com/mamedev/mame.git mame_src
cd mame_src
git sparse-checkout init --no-cone
git sparse-checkout set 'src/mame/**/*.cpp'
git checkout
```

~150 MB, ~4,660 files, a few seconds. From there, regex-extract every
`GAME(`/`GAMEL(` macro's `(setname, parent, ...)` across the whole tree (31.7k
rows as of this writing) into a local table — **don't anchor the setname
pattern to start with a letter**, MAME has plenty of digit-leading setnames
(`99lstwar`, `4dwarrio`, `280zzzap`) and a naive `[A-Za-z_]` anchor will
silently drop them and produce false "missing" findings. Then check every
mister-devel/alternatives setname for existence in that table.

**Two false-positive traps, learned the hard way doing this the first time —
see [`reports/2026-08-06-full-mame-crossref-audit.md`](reports/2026-08-06-full-mame-crossref-audit.md)
for the full writeup:**

1. **`<parent>` tag mismatches are not a reliable signal.** MiSTer `.mra`s
   bundle full ROM content and use the `<parent>` tag inconsistently
   (sometimes self-referential for root sets, often blank for real MAME
   clones). Only trust a *missing setname*, not a parent-tag diff, as a
   finding on its own.
2. **Most of `MRA-Alternatives_MiSTer`'s "missing" setnames are HBMAME, not
   MAME.** HBMAME is a separate hack/trainer/homebrew fork — its content was
   never in mainline MAME and never will be. The filename reliably says so
   (contains `HBMame`, e.g. `Donkey Kong Foundry - HBMame.mra`) — filter
   these out before treating anything as a lead. In the 2026-08-06 pass this
   cut 223 raw "missing" entries down to 16 worth actually looking at.

Save the extraction as `data/mame-current-sets.tsv` when you do this — same
staleness caveat as every other snapshot in this pack.

## Open findings (carry forward until fixed)

Standing to-do list, independent of which MAME version is current. **Check
this at the start of every run**, not just the changelog gap — these were
found once and nothing here re-verifies them automatically. For each, a
quick check is enough: pull the live `.mra` (mister-devel.md's method) and
see whether it still shows the old setname. If it's been fixed upstream since
the last audit, mark it fixed in the entry below that raised it and drop it
from this list; don't silently re-report it as new.

**Confirmed renames, exact fix known, not yet applied (as of 2026-08-06):**
`twinbeeb`→`bs_twinbee` (Arcade-BubSys_MiSTer), `devilfsg`→`devilfshg`
(Arcade-Galaxian_MiSTer), `gauntletr8`→`gauntletgr8` (Arcade-Gauntlet_MiSTer
**and** MRA-Alternatives), `imgfightb`→`imgfightjb` (Arcade-IremM72_MiSTer),
`popflamn`→`popflamen` (Arcade-NaughtyBoy_MiSTer), `SpaceDemon`→`spacedem`
(Arcade-SpaceFirebird_MiSTer), `kengoa`→`kengoj` (Arcade-IremM72_MiSTer),
`pengo2`→`pengoa`, `pengo4`→`pengoc`, `pengo5`→`pengob` (all
MRA-Alternatives `_Pengo`), `bagmans2`→`bagmans4` (MRA-Alternatives `_Bagman`),
`joustwr`→`jousty` (MRA-Alternatives `_Joust`), `sinistar1`→`sinistarp`
(MRA-Alternatives `_Sinistar`). The last 7 were resolved by walking MAME's git
history (see "Rename archaeology" below) — none are 0.289-specific, all
predate this pack. Detail, exact commit hashes, and exact ROM/parent values
in [`reports/2026-08-06-full-mame-crossref-audit.md`](reports/2026-08-06-full-mame-crossref-audit.md).

**Could not confirm despite a full git-history search on the owning driver
file — likely never an official MAME setname, not pursued further:**
`spclone`, `spcloneo` (Arcade-Salamander_MiSTer / MRA-Alternatives
`_Salamander`), `ironhorsbl` (MRA-Alternatives `_Iron Horse`). Don't
re-search these from scratch on a future run unless something new comes up —
same report has what was already tried.

**Not tracked — checked once, deliberately dropped, don't re-flag:** fan
hacks/patches with no official MAME romset behind them
(`rtype2inv`, `xmultiplm72inv`, `cleansweept`, `mrdonight`, `tutankhm2`,
`athenaff`, `ddonpachjt`, `esprade_fp`, `espradej_fp`), content MAME doesn't
emulate at all (`spacerace` — discrete TTL, no ROMs), and all
HBMAME-sourced `MRA-Alternatives_MiSTer` content (filename contains
`HBMame`). See the report's "What's out of scope, on purpose" section before
re-adding anything like this.

**Cosmetic, low priority, not blocking:** stale `<mameversion>` tags on the
non-Europe Sailor Moon alternatives (still `0275`, content confirmed correct)
and on the `nslasher` family (still `0264`) — see the 0.289 report.

### Rename archaeology (git history, not just current-state comparison)

For setnames that don't exist in current MAME with no obvious replacement,
don't stop at "needs manual review" — check whether it's a rename MAME made
at some point in the past that just never got picked up (this is exactly the
TwinBee situation, minus a changelog line to point at it). `git log -S` on a
local partial clone of MAME's full history answers this directly and is fast
enough to use routinely on the handful of setnames that reach this point:

```bash
git clone --filter=blob:none --no-checkout https://github.com/mamedev/mame.git mame_hist
# ~210 MB, ~9 seconds — full commit graph + trees, no blobs, no --depth limit
cd mame_hist
git log -S"<old-setname>" --follow --oneline -- src/mame/<mfg>/<driver>.cpp
```

Use `--filter=blob:none`, not `--filter=tree:0` — trees need to stay local for
`git log` traversal to be fast (tree:0 was tried first and made every step a
slow network round-trip). `--follow` is required, not optional: MAME's 2022
driver reorganization moved most files from `src/mame/drivers/*.cpp` into
manufacturer subfolders, and renames worth finding often predate that move —
`--follow` walks through the reorg transparently, a plain path-restricted
search won't. When `-S` finds a hit, `git show <sha> -- <path>` gets you the
diff; if the file was at a different historical path at that commit (check
with `git show --stat <sha>`), re-run `git show` against that path instead —
the given `<path>` has to match the tree at that specific commit.

Confirm any hit by reading the actual diff (old ROM comment/description text
usually carries over verbatim or near-verbatim to the new entry — that's the
match, not just proximity in the commit). A handful of setnames in this pack
searched their driver's entire tracked history (back to MAME's original git
import, ~2008) and found nothing — treat that as reasonably strong evidence
the setname was never official MAME, not as a search failure to retry.

## Run history

Update this after every audit. Newest first.

---

### Full cross-reference audit (not tied to a single MAME version) — 2026-08-06

Same day as the 0.289 audit below, separate pass: instead of reading a
changelog, checked every setname in mister-devel + `MRA-Alternatives_MiSTer`
(1,848 total) for existence and parent-match against a full local clone of
current MAME source (`data/mame-current-sets.tsv`, 31,782 machine
definitions). This is what raised everything in "Open findings" above. Full
writeup: [`reports/2026-08-06-full-mame-crossref-audit.md`](reports/2026-08-06-full-mame-crossref-audit.md).

Summary: 15 core-repo setnames and 16 non-HBMAME alternatives setnames don't
exist under those names in current MAME. Followed up same day with git
history archaeology (`git log -S`, see "Rename archaeology" above) on the 10
that had no obvious replacement: **7 turned out to be real MAME renames**,
confirmed against the actual diff, dates ranging 2020–2023 — none
0.289-specific. 3 (`spclone`, `spcloneo`, `ironhorsbl`) found nothing across
their driver's entire tracked history and are presumed never-official. The
remainder were fan hacks/patches or HBMAME content (207 of the 223 raw
alternatives misses) and are deliberately not tracked — see "Open findings"
above for the current breakdown and the report for full detail.

None of this is 0.289-specific — most of it looks considerably older, i.e.
it's drift that accumulated across releases before this pack existed. This
kind of full pass is worth repeating occasionally (not necessarily every
run, it's a heavier pull) since the changelog-only method can't catch
anything that isn't in the version range being read. Re-run it in full again
around every 10-20 changelog-driven audits, or whenever asked for a "full"
check specifically rather than "what's new."

---

### 0.289 (2026-07-31) — audited 2026-08-06

**Scope:** full read of `whatsnew_0289.txt`, cross-checked against a full
snapshot of all 172 `Arcade-*_MiSTer` repos and `MRA-Alternatives_MiSTer`
(~1,850 `.mra` files total), with leads verified against live MAME source.
Full writeup: [`reports/2026-08-06-mame-0289-audit.md`](reports/2026-08-06-mame-0289-audit.md).

Summary of findings:
- **Confirmed, unfixed:** `Arcade-BubSys_MiSTer`'s TwinBee (Bubble System) `.mra`
  still targets the pre-0.289 setname `twinbeeb`; MAME 0.289 renamed it to
  `bs_twinbee` and reparented it under a new `bubsys` BIOS root. Needs an
  `.mra` rebuild.
- **Already fixed:** `Arcade-CaveBanpresto_MiSTer`'s main Sailor Moon `.mra`
  already reflects 0.289's `sailormn`-is-now-Japan reparent correctly
  (`mameversion` bumped, correct zip/CRCs). `MRA-Alternatives_MiSTer`'s Europe
  variants (all three date-revisions) were also already updated. The
  non-Europe alt variants (Japan/US/HK/KR/TW × 3 date-revisions) are still
  functionally correct — their setnames didn't move — but their
  `<mameversion>` tags are stale at `0275`, and their `.mra` content wasn't
  re-verified against current CRCs in this pass.
- **New finding, this run:** `ArcadeDatabase_MiSTer/ArcadeDatabase.csv` still
  describes setname `sailormn` as the Europe version — stale since 0.289's
  reparent.
- **Verified as non-issues:** the `dataeast/deco32.cpp` split (0.289) and the
  `dataeast/deco156.cpp` → `hvysmsh.cpp` rename don't change any setnames
  MiSTer currently ships (`nslasher` family confirmed unchanged in the new
  driver file); the King of Boxer and Shikigami no Shiro changes don't touch
  any game MiSTer currently covers at all.
- Several "new working clone" entries (Robotron 2084 Release 3 prototype,
  Pleiads ManilaMatic bootleg, Galaxian Olympia bootleg, Turbo Force Japan,
  Ninja Ryukenden set 3, Konami RF2 bubble-system promoted to working) are
  **opportunities**, not breakage — candidates for new `MRA-Alternatives_MiSTer`
  entries, not yet acted on.

Next run should start from **0.290** (whatever the changelog file is named
when it ships) and also do a light CRC spot-check on the still-flagged Sailor
Moon alt variants above if that wasn't already resolved by hand.
