# Full audit: every mister-devel `.mra` vs. current MAME source — 2026-08-06

Unlike the [changelog-driven 0.289 audit](2026-08-06-mame-0289-audit.md), this
pass doesn't rely on reading release notes at all. It cross-references **every
setname in mister-devel and `MRA-Alternatives_MiSTer` against MAME's current
source directly** — catching drift accumulated over any number of past
releases, not just the latest one.

## Method (new — folded into [mame.md](../mame.md))

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

Worth recording prominently because they'll otherwise produce a wall of false
positives on every future run:

### 1. `<parent>` mismatches are not a usable signal on their own

886 of 1,848 entries "mismatched" on parent by naive comparison — but manual
inspection showed this is overwhelmingly a MiSTer authoring convention, not
drift. MiSTer `.mra` files bundle complete ROM data directly (unlike MAME's
merged/split zip inheritance), so the `<parent>` tag is optional documentation,
filled in inconsistently: some root/parent releases self-reference their own
setname in `<parent>` instead of leaving it blank; many real MAME clones ship
with no `<parent>` tag at all. **Don't treat a parent-tag mismatch as a finding
by itself** — it's only meaningful together with a setname that's actually
missing (see the TwinBee/Sailor Moon cases in the other report, which were
real because the *setname* moved, not because a parent tag was "wrong").

### 2. Most of `MRA-Alternatives_MiSTer`'s "missing" setnames are HBMAME, not MAME

207 of 223 "missing" setnames in `MRA-Alternatives_MiSTer` turned out to be
[HBMAME](https://hbmame-roms.com/) content — a separate hack/homebrew/trainer
fork of MAME, not mainline MAME. These were never going to match
`mamedev/mame` source and this isn't drift. **The tell is reliable and
mechanical: the `.mra` filename itself contains `HBMame`** (e.g. `Donkey Kong
Foundry - HBMame.mra`). Filter these out by filename before treating anything
as a finding — it cuts the alternatives "missing" list from 223 to 16.
Several of the remaining 16 also turned out to be non-MAME (fan patches like
"Free Play" or "(inv)"/invincibility hacks) once inspected individually —
filename alone doesn't catch all of it, but it removes the bulk.

## Findings — `Arcade-*_MiSTer` core repos (15 setnames not found in current MAME)

| Setname (MiSTer) | Repo / file | Status |
|---|---|---|
| `twinbeeb` | Arcade-BubSys_MiSTer | **Confirmed rename** → `bs_twinbee`, parent `bubsys`. Same as the 0.289 report — still unfixed. |
| `devilfsg` (parent declared: `devilfish`) | Arcade-Galaxian_MiSTer | **Confirmed rename**, `src/mame/galaxian/galaxian.cpp` — current names are `devilfshg` (clone) / `devilfsh` (parent). Not 0.289-specific; predates it. |
| `gauntletr8` (parent: `gauntlet`) | Arcade-Gauntlet_MiSTer | **Confirmed rename** → `gauntletgr8` (German-revision sets carry a `g` in the setname: `gauntletgr8`, `gauntletgr6`, `gauntletgr3`). |
| `imgfightb` | Arcade-IremM72_MiSTer | **Confirmed rename** → `imgfightjb` (`src/mame/irem/m72.cpp`). |
| `popflamn` (parent declared: `popflamn`, self-referential) | Arcade-NaughtyBoy_MiSTer | **Confirmed rename** → `popflamen`, parent `popflame` (`src/mame/phoenix/naughtyb.cpp`). |
| `SpaceDemon` (parent declared: `SpaceFirebird`) | Arcade-SpaceFirebird_MiSTer | **Confirmed — mis-cased/never matched MAME naming.** Current MAME: setname `spacedem`, parent `spacefb` (`src/mame/nintendo/spacefb.cpp`). Both the setname and parent in this `.mra` use the MiSTer core's display name instead of MAME's actual lowercase setnames — this doesn't look like a recent MAME rename so much as this `.mra` never having matched MAME's real naming. |
| `kengoa` | Arcade-IremM72_MiSTer | **Needs manual review.** No `kengoa` in current `m72.cpp`; current clones under parent `ltswords` are `kengo` (World) and `kengoj` (Japan). Unclear which one "(set 2)" is meant to map to — worth checking MiSTer's actual ROM contents against both before renaming. |
| `spclone`, `spcloneo` ("Salamander SP version"[/"old"]) | Arcade-Salamander_MiSTer | **Needs manual review.** Not found under any name in current `src/mame/konami/nemesis.cpp`'s Salamander family (`salamand`, `salamandj`, `salamandt`, `lifefrce`, `lifefrcej`). Unclear if this was ever an official MAME set or if it's misnamed/removed. |
| `rtype2inv`, `xmultiplm72inv` | Arcade-IremM72_MiSTer | **Likely not a MAME set at all.** `(Inv)` strongly suggests an invincibility-patched hack, not an official romset — not found under any name in `m72.cpp`. Low priority; verify before spending time on it. |
| `cleansweept`, `mrdonight` | Arcade-Galaxian_MiSTer | **Likely not a MAME set at all.** No trace of "Clean Sweep" or "Mr. Do Nightmare" in any driver; both ship with `<mameversion>0000</mameversion>` (never set), consistent with Galaxian-hardware fan hacks that were never based on a MAME dump. |
| `tutankhm2` | Arcade-Tutankham_MiSTer | **Likely not a MAME set at all.** "Tutankham II" reads as a fan sequel/hack; not present in any driver. |
| `spacerace` | Arcade-SpaceRace_MiSTer | **Not applicable.** This is a 1973 Atari discrete-logic (TTL, no ROMs) game. It's *referenced* in `src/mame/atari/atarittl.cpp`'s documentation table but has no `GAME()` entry there or anywhere else in current MAME — it isn't emulated by mainline MAME at all. MiSTer's implementation is independent of MAME here; there's nothing to sync against. |

## Findings — `MRA-Alternatives_MiSTer` (16 non-HBMAME "missing" setnames, of 223 total)

| Setname | Folder | Status |
|---|---|---|
| `gauntletr8` | `_Gauntlet` | Same fix as the core-repo entry above — `gauntletgr8`. This setname is stale in *two* places. |
| `pengo2`, `pengo4`, `pengo5` | `_Pengo` | **Confirmed old naming scheme.** Current `src/mame/pacman/pengo.cpp` uses lettered clones (`pengoa`, `pengob`, `pengoc`, `pengoja`, `pengojb`...), not numbered ones. This predates 0.289 by a wide margin — worth a review pass to map each old numbered file to its current lettered equivalent by ROM content, not just by number. |
| `bagmans2` | `_Bagman` | **Needs manual review.** `src/mame/valadon/bagman.cpp` currently has `bagmans` (Stern rev A5), `bagmans3` (rev A3), `bagmans4` (rev A4) — no `bagmans2`. Likely an old/consolidated revision naming. |
| `ironhorsbl` | `_Iron Horse` | **Needs manual review.** `src/mame/konami/ironhors.cpp` currently has `ironhors`, `ironhorsh`, `dairesya`, and `farwest` ("bootleg?") — no `ironhorsbl`. Possibly consolidated into `farwest`. |
| `joustwr` | `_Joust` | **Needs manual review.** `src/mame/williams/williams.cpp` currently has `joust` (Green label), `joustr` (Red label), `jousty` (Yellow label) — no "White-Red" variant. |
| `sinistar1` | `_Sinistar` | **Needs manual review.** Current Sinistar family: `sinistar` (rev 3 upright), `sinistarc` (rev 3 cockpit), `sinistar2`/`sinistarc2` (rev 2), `sinistarp` (AMOA-82 prototype) — no `sinistar1`. |
| `athenaff` | `_Athena` | **Likely not a MAME set.** Filename is literally `Athena_Screen_Flip_Fix_ROM_Patch.mra` — a community ROM patch, not an official dump. |
| `ddonpachjt` | `_DoDonPachi` | **Likely not a MAME set.** "Trainer" = cheat-patched hack. |
| `esprade_fp`, `espradej_fp` | `_ESP Ra.De` | **Likely not a MAME set.** "(Free Play)" fan patches. |
| `rtype2inv`, `xmultiplm72inv`, `spclone`, `spcloneo` | various | Same entries as the core-repo table above — these exist in both places. |

The other 207 `MRA-Alternatives` "missing" setnames are HBMAME hack content
(filename contains `HBMame`) and are expected to not match mainline MAME —
not findings. Examples for context, not action: `dkong2m`/`dkongpac`/`dkrdemo`
("Donkey Kong ... - HBMame.mra"), `astroped`/`killiped`/`vectiped` (Centipede
hacks), `galagaef`/`galagost`/`vgalaga` (Galaga hacks).

## What wasn't checked

- **CRC/content-level verification** was only done for entries flagged above
  as suspicious, plus the two cases already confirmed in the 0.289 report
  (Sailor Moon's non-Europe alt variants were spot-checked and their CRCs do
  match current source despite the stale version tag). Entries not listed
  above (~1,600 of 1,848) matched on setname; their exact ROM part CRCs
  weren't individually re-verified against source — that's a much larger
  parsing effort (MAME ROM_LOAD macros are frequently shared via `#define`
  blocks across multiple sets, which is easy to mis-parse) and wasn't judged
  worth it without a specific reason to suspect a given set.
- Jotego-framework cores are still out of scope (see mister-devel.md).

## Priority summary

**Fix now (confirmed, exact new values known):** `twinbeeb`→`bs_twinbee`,
`devilfsg`→`devilfshg`, `gauntletr8`→`gauntletgr8` (both locations),
`imgfightb`→`imgfightjb`, `popflamn`→`popflamen`, `SpaceDemon`→`spacedem`.

**Investigate (setname confirmed gone, replacement unclear):** `kengoa`,
`spclone`/`spcloneo`, `pengo2`/`pengo4`/`pengo5`, `bagmans2`, `ironhorsbl`,
`joustwr`, `sinistar1`.

**Not MiSTer/MAME sync issues — no action:** everything tagged "likely not a
MAME set" or "not applicable" above, and all 207 HBMAME-sourced alternatives.
