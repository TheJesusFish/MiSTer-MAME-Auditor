# `mra_rom_check.sh` re-sweep — 2026-08-23

> **Correction, added same day:** this report's "121 failures across 9
> repos, none excluded by `MRA-Alternatives_MiSTer`" claim was wrong — the
> exclusion filter wasn't actually applied correctly against the initial
> sweep. Re-checked by explicit `comm -12` between sorted setname lists
> (fresh `MRA-Alternatives_MiSTer` clone, 853 setnames, 858/858 files
> independently verified passing): every failing setname in
> `Arcade-ActFancer_MiSTer` (3), `Arcade-BoogieWings_MiSTer` (4),
> `Arcade-NightSlashers_MiSTer` (3), and `Arcade-TrioThePunch_MiSTer` (1)
> was already correctly tracked there and needed no fix at all — same for
> 67 of `Arcade-IGSPGM_MiSTer`'s 68 alternates (only `puzzli2s` is a real
> gap) and 29 of `Arcade-AtariSystem2_MiSTer`'s 34 failures (only the 5 true
> root/parent setnames — `720`, `apb`, `csprint`, `paperboy`, `ssprint` —
> are genuinely outside that repo's scope). The local fixes already made to
> all of these are still factually correct, just mostly unnecessary; nothing
> below is retracted otherwise. See mame.md's Open Findings for the
> corrected, current-state breakdown — this file is left as the historical
> record of what the sweep actually found and did that day.

## Ask

1. Re-run the full `mra_rom_check.sh` sweep across all `Arcade-*_MiSTer` core
   repos since the last pass (2026-08-06), scoped to `releases/` only per the
   rule added to prompt.md/mister-devel.md in the meantime.
2. Specifically resolve what
   [`Arcade-IGSPGM_MiSTer` commit `eae22c91`](https://github.com/MiSTer-devel/Arcade-IGSPGM_MiSTer/commit/eae22c91f58b3d1f057bbf158c15234a723bed70)
   represents, and whether anything postdates it.

## Part 1: the `eae22c91` commit

Resolved via the GitHub API and a direct content check, not assumption:

- The commit is authored by the user (`TheJesusFish`), titled "MRA
  Housekeeping", dated 2026-08-07. Its parent is `cb193301d9ea8ef3c284d8e0c7d2893c46355642`
  ("Release 20260707"), which **is** the actual current tip of upstream
  `main` as of this check.
- Its diff modifies exactly 2 files:
  `releases/Puzzle Star (ver. 100MG, WORLD).mra` and
  `releases/Puzzli 2 Super (ver. 200, WORLD).mra`. The diff content is
  identical to the fix already applied and recorded in this pack's
  2026-08-06 run: adding the 3 shared PGM BIOS CRCs
  (`pgm_p02s.u20`=`78c15fa2`, `pgm_t01s.rom`=`1a7123a0`,
  `pgm_m01s.rom`=`45ae7159`) and, for the Puzzli 2 Super file, fixing
  `zip="pgm.zip|puzzli2s.zip"` → `zip="pgm.zip|puzzli2.zip|puzzli2s.zip"`
  (a missing intermediate parent in the pgm→puzzli2→puzzli2s chain).
- **But `eae22c91` is not the tip of `main` or `dev`** — GitHub's
  `commits/{sha}/branches-where-head` API returns empty for it, and neither
  branch descends from it. It's a dangling/orphaned commit: pushed to the
  repo's object store (visible by SHA, via `.patch`, via the commit API) but
  never actually merged into any branch anyone builds from.
- Confirmed directly: a fresh clone of `main` (HEAD `cb19330`) still has
  **zero** `crc=` attributes on those 3 BIOS parts in both files, and the
  Puzzli 2 Super `zip=` still reads the old, incomplete
  `"pgm.zip|puzzli2s.zip"`. Whatever this commit was meant to be — a PR that
  never landed, a push to a since-deleted branch — its content never reached
  the live default branch.

**Conclusion: nothing postdates `eae22c91` in the sense of "newer changes to
check" — the opposite is true. The fix it represents was never live to begin
with.** Re-applied it fresh to a new local clone (the repo's earlier local
clone from the 2026-08-06 work was gone from disk) rather than trying to
recover or push the orphaned commit itself.

## Part 2: full re-sweep

Repo count is now **182** (was 172 on 2026-08-06); `Arcade-AtariSystem2_MiSTer`
is new, added 2026-08-20. Fresh sparse-clone of all 182 repos scoped to
`releases/*.mra` + `releases/**/*.mra` (per the `releases/`-only editing rule
— `docs/` and other non-`releases/` `.mra` files are out of scope for fixing,
though the sweep tooling itself only pulled `releases/` this time so there's
nothing to separately exclude). Fresh full clone of `MRA-Alternatives_MiSTer`
(817 files, up from 808) for the setname exclusion list (812 unique
setnames).

**121 failures across 9 repos** (86 missing-CRC entries, 35 missing-`<mameversion>`
entries), none matching a setname already tracked in `MRA-Alternatives_MiSTer`:

| Repo | Failures | Cause |
|---|---|---|
| `Arcade-ActFancer_MiSTer` | 3 | missing CRCs, `releases/alternatives/` |
| `Arcade-AtariSystem2_MiSTer` | 34 | missing `<mameversion>` (all files, new repo) |
| `Arcade-BoogieWings_MiSTer` | 4 | missing CRCs, `releases/alternatives/` |
| `Arcade-IGSPGM_MiSTer` | 69 | missing CRCs (2 flagship + 67 more in `releases/_alternatives/`) |
| `Arcade-KickAndRun_MiSTer` | 1 | missing `<mameversion>` |
| `Arcade-NightSlashers_MiSTer` | 3 | missing CRCs, `releases/_alternatives/_Night Slashers/` |
| `Arcade-SNK6502_MiSTer` | 3 | missing CRCs (2 of which were masking a merge-conflict bug — see below) |
| `Arcade-Sonson_MiSTer` | 1 | missing CRCs |
| `Arcade-TrioThePunch_MiSTer` | 1 | missing CRCs, `releases/alternatives/` |

### Why several "already fixed" repos still showed failures

Six of these nine repos (all but ActFancer and AtariSystem2) had a fix
already recorded from the 2026-08-06 run. Two different situations, not one:

**A. New sibling files, not a regression (BoogieWings, NightSlashers,
TrioThePunch).** The 2026-08-06 fix targeted one flagship file per repo
(Euro/Korea/World respectively). Those fixes *are* live upstream — merged via
PR #1 in each repo. This sweep's failures are on different files entirely:
regional/revision alternates (BoogieWings' Asia/USA/both Ragtime-Japan
variants; NightSlashers' Over Sea/US/Japan variants; TrioThePunch's Japan
variant) that share the same missing-CRC problem but were never touched
before, because the earlier pass's check didn't reach them.

**B. Fix commits that exist locally but were never pushed (KickAndRun,
Sonson, SNK6502).** These three had a `Fix MRA(s)` commit sitting in the
local clone from 2026-08-06, and `git status` reported the local branch as
"up to date with origin" — which is true but misleading: it means the local
branch matches its *own* remote-tracking ref, not that the commit made it
into the actual upstream default branch other people build from. A fresh
independent clone confirmed the fix was never live. (This is the same shape
of problem as the `eae22c91` mystery in Part 1 — a fix that exists somewhere
in git but never reached the branch that matters.)

### The SNK6502 merge-conflict bug

While re-verifying repo B above, `Arcade-SNK6502_MiSTer/releases/Vanguard.mra`
and `Fantasy.mra` failed with "broken XML: not well-formed" — not the
missing-CRC error the original sweep found, a new one. Both files had
**unresolved `<<<<<<< HEAD` / `=======` / `>>>>>>> 9540935...` git conflict
markers committed directly into the XML**, left over from whenever the
2026-08-06 `Fix MRAs` commit was made. Easy to miss on a casual read — the
file still looks roughly like valid XML around the markers — but
`mra_rom_check.sh` catches it immediately as a parse failure. Resolved by
keeping the CRC'd side of each conflict and discarding the markers. While in
there, also found and filled 6 CRCs that **neither side** of either conflict
had ever populated — Vanguard's and Fantasy's HD38880 speech-ROM parts
(`sk6_ic07/08/11.bin`, `fs_d_7/e_8/f_11.bin`) — confirmed against current
MAME driver source (`src/mame/snk/snk6502.cpp`).

### Fixes applied

All 121 original failures plus the 2 newly-discovered SNK6502 issues, fixed
in local clones under `/Users/thejesusfish/Documents/GitHub/`:

- **CRC additions**: matched against current MAME driver source
  (`dataeast/actfancr.cpp` for ActFancer/TrioThePunch, `dataeast/boogwing.cpp`
  for BoogieWings, `dataeast/fghthist.cpp` for NightSlashers,
  `snk/snk6502.cpp` for SNK6502, `igs/pgm.cpp` family for IGSPGM's shared
  BIOS parts). Where a `.mra` uses the `setname/filename` convention for
  clone-specific parts vs. bare `filename` for parent-shared parts (the
  BoogieWings/NightSlashers/TrioThePunch alternates all do this, with Italian
  comments in the `.mra` explicitly documenting which parts are shared vs.
  clone-specific), matched accordingly rather than guessing.
- **`<mameversion>0289</mameversion>`** inserted after `<setname>` in all 34
  `Arcade-AtariSystem2_MiSTer` files and the 1 KickAndRun file — `0289`
  matches this pack's established convention (used identically for Freeze,
  MrJong, KickAndRun's original 2026-08-06 fix, etc.), chosen because
  AtariSystem2 is new enough (2026-08-20) that it has no prior convention of
  its own to match.
- **IGSPGM's 67-file blanket fix**: same 3 shared BIOS CRCs
  (`pgm_p02s.u20`/`pgm_t01s.rom`/`pgm_m01s.rom`) applied wherever those part
  names appeared without a `crc=` attribute anywhere under `releases/`, not
  just the 2 originally-flagged files — the entire PGM lineup in this repo's
  `releases/_alternatives/` folder was missing the same CRCs, confirmed by
  the check script rather than assumed. A regex guard skipped any tag that
  already carried a `crc=` (a handful of DoDonPachi III files already had
  the `pgm_p02s.u20` CRC filled in, just not the other two) to avoid
  producing duplicate attributes.
- **SNK6502 conflict resolution + speech-ROM CRCs**, as described above.

Every touched repo re-verified 100% passing with `mra_rom_check.sh -ir`
after the fix. All diffs checked with `git diff --stat` for symmetric
insertions/deletions (no CRLF line-ending corruption). Nothing beyond
`<mameversion>`/CRC additions and the conflict-marker cleanup was touched —
no `<about>` notes, no unrelated edits.

**Nothing was staged, committed, or pushed** — per the standing instruction,
every fix above is an uncommitted working-tree change in a local clone,
ready for the user to review and commit/push through their own workflow.
