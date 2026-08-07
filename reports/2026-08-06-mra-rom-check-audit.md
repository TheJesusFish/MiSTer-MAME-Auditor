# `mra_rom_check.sh` sweep across all `Arcade-*_MiSTer` core repos — 2026-08-06

List only, per instruction — nothing fixed in this pass.

## Method

Ran MiSTer-devel's own validator
([`Scripts_MiSTer/other_authors/mra_rom_check.sh`](https://github.com/MiSTer-devel/Scripts_MiSTer/blob/master/other_authors/mra_rom_check.sh))
against every `.mra` in every `Arcade-*_MiSTer` repo (172 repos, refreshed
live; 168 publish `.mra` content — same 4 known-empty/Jotego-pattern repos as
always, see mister-devel.md). Pulled via the same rate-limit-free sparse-clone
technique used throughout this pack (`*.mra`/`**/*.mra` sparse-checkout
pattern, ~93 MB for 1,045 files across 172 repos, no `.rbf` binaries).

Ran with `--ignore-roms` — no local MAME romset to check CRCs against real
dumped bytes with, and that's not something to source (copyrighted ROM
data). This still exercises everything else the script checks: XML
well-formedness, `<mameversion>` tag presence, and that every ROM `<part>`
with a `name` attribute also has a `crc` attribute. `MRA-Alternatives_MiSTer`
was already swept this way separately and came back 808/808 clean — this
pass is the core repos only.

**Result: 1,045 checked, 884 clean, 161 flagged across 14 repos.**

**Refined down to 81 across 13 repos** by dropping anything whose `<setname>`
is also currently present in `MRA-Alternatives_MiSTer` — that repo was
already swept and any fixable issues there are handled separately, so this
list is scoped to problems in a core repo's own content, not a duplicate of
something living in (or already fixed in) the alternatives repo. Matched by
setname, not by folder name (`_alternatives`/`Alternative Sets`/`docs/`
folders inside a core repo aren't reliably 1:1 with the actual
`MRA-Alternatives_MiSTer` repo — e.g. `Arcade-IGSPGM_MiSTer`'s own internal
`_alternatives` folder covers different games entirely, so most of its 70
flagged files stayed dropped for being real duplicates of tracked
alternatives content, but 3 remained because those specific setnames
(`puzzli2s`, `puzlstar`) aren't tracked over there at all).

Category meanings and the investigation into what's confirmed vs. still an
open question are below, from the original 161-file pass — still accurate
as background, but the counts it references are from the first pass; the
numbers actually current are in the re-scan section right below.

## Re-scan — same day, after upstream fixes landed

Re-pulled all 172 repos (`git fetch --depth 1` + `reset --hard` on the
existing sparse clones, not full re-clones — a few seconds per repo) and ran
the same check again, since some maintainers had already started fixing
things independently of this pack (noticed because `Arcade-DECOCassette_MiSTer`'s
latest pull came in on a commit literally titled "Fixed MRAs - XML and
CRCs"). Re-filtered against a refreshed `MRA-Alternatives_MiSTer` setname
list the same way as before.

**Down to 22 across 12 repos, from 81 across 13 — 59 resolved:**

- `Arcade-Kyugo_MiSTer` — fully resolved (was 5, all missing-`<mameversion>`).
- `Arcade-DECOCassette_MiSTer` — 55 → 1. Only
  `Alternative Sets/Ocean to Ocean (Japan) (DECO).mra` (`cocean1a`, broken
  XML — same illegal `--`-in-comment issue) remains; the other `Ocean to
  Ocean` file and `Flash Boy` (the other two broken-XML cases) and all 52
  missing-`<mameversion>` files got fixed in the same pass.
- Everything else on the list is unchanged from the first pass — see
  [`data/mra-rom-check-failures.md`](../data/mra-rom-check-failures.md) for
  the current 22-file list (that file now reflects the re-scan, not the
  original 81).

## Confirmed real: broken XML (4 files)

- **`Arcade-IremM72_MiSTer` — `docs/irem_m84_mra/Cosmic Cop (World).mra`**:
  the opening `<rom index="0" ...>` tag is missing entirely — ROM data
  (`<interleave>`/`<part>` entries) starts right after `</switches>` with no
  wrapping tag, but a `</rom>` closer still appears further down with nothing
  to close. Genuinely malformed, confirmed by inspecting the file directly
  (verified with `grep -n '<rom\|</rom>'` — the first `<rom` open tag in the
  file comes *after* the orphaned `</rom>` close).
- **`Arcade-DECOCassette_MiSTer` — 3 files** (`Ocean to Ocean (DECO).mra` ×2,
  `Flash Boy (DECO).mra`): a comment contains a literal `--` inside its body
  — `<!-- ... rms-3_p2-.c9, 1KB @ 0xFC00) -- OLD-AUDIO-PROM-FIX-2026-06-29 -->`
  — which is illegal in strict XML (a comment can't contain `--` anywhere
  except immediately before the closing `-->`). The `2026-06-29` in the
  comment text suggests this was introduced by a recent edit.

## Confirmed real, unambiguous: missing `<mameversion>` (66 files)

No ambiguity here — the tag is either present or it isn't. Concentrated:
**50 of 66 are in `Arcade-DECOCassette_MiSTer`** (nearly its entire
`releases/` and `Alternative Sets/` folders), plus `Arcade-Kyugo_MiSTer` (7),
`Arcade-Freeze_MiSTer` (5), and one each in `Arcade-KickAndRun_MiSTer` and
`Arcade-MrJong_MiSTer`. Full list in the data file linked above.

## Needs judgment, not unambiguous: missing CRCs on named `<part>`s (94 files)

The checker flags any `<part>` that has a `name` attribute but no `crc`
attribute. Investigated the actual XML behind this rather than take the
count at face value, since the script doesn't understand every legitimate
`.mra` construct — found two different situations that look the same in the
script's output but aren't the same kind of problem:

**70 files, all in `Arcade-IGSPGM_MiSTer`, same pattern, likely a real
gap:** every flagged file is in `releases/_alternatives/`, and every one is
missing the *same* three CRCs, for the *same* three shared BIOS ROM files
(`pgm_p02s.u20`, `pgm_t01s.rom`, `pgm_m01s.rom`). Checked a primary
(non-alternate) release in the same repo (`Martial Masters (ver. 104, 102,
102US).mra`) — it declares all three, with real values
(`78c15fa2`/`1a7123a0`/`45ae7159`). So the primary release has these CRCs and
every alternate is consistently missing exactly the same three — reads like
the alternates were built from a template that dropped the shared-BIOS CRCs,
not 70 independent oversights. Not fixed here (per instruction), but the
correct values are already known if this gets addressed later.

**~24 files elsewhre (BoogieWings, NightSlashers, SNK6502, ActFancer,
Sonson, TrioThePunch, Tutankham): genuinely unclear.** Some of these use
`offset=`/`length=`/`map=` attributes to slice a chip's data across multiple
`<part>` entries (e.g. Act-Fancer's interleave block references parts named
`"15"`/`"16"`/`"00"`/`"01"` this way) — reasonable to assume the CRC only
needs to live on one instance and the checker just doesn't do that
cross-referencing. **But checked, and that's not actually what's
happening**: in every case checked here, *none* of the same-named part
instances carry a `crc` anywhere in the file — not "the checker missed a
sibling with a crc," the crc is absent everywhere for that chip. Whether
that's intentional (this MRA format doesn't require a CRC on offset/length
slices by design) or a real gap wasn't resolved — didn't have a clean way to
confirm which, and didn't want to guess. One useful cross-check:
`Arcade-Tutankham_MiSTer/Alternative Sets/Tutankham II.mra` — `tutankhm2` —
is on this list, and that's the same fan-hack setname already flagged and
deliberately dropped in the very first audit in this pack (not an official
MAME romset at all) — consistent with it not having real CRCs, for an
unrelated reason than the others on this list.

## Not evaluated

Byte-level CRC-vs-real-dump verification (needs an actual MAME romset,
intentionally not sourced here) and the `zip=` completeness question (a
separate, already-completed sweep — see mame.md's "Checking `zip=`
completeness" section and the 2026-08-06 report — that one *was* MRA-Alternatives-only;
whether the same gap exists across the core repos hasn't been checked).
