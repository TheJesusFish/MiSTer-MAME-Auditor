# mister-devel.md — the MiSTer side of the audit

This file describes **where MiSTer's arcade ROM definitions live**, how to refresh
that picture before every audit, and what's already known about how it's organized.
Read [prompt.md](prompt.md) first for the overall task; read [mame.md](mame.md) for
the MAME side.

## The three repos that matter

MiSTer arcade content isn't in one place. A single game's romset can be defined
in any of these three, and MAME changes can break any of them independently:

| Repo | What it holds | Org |
|---|---|---|
| `Arcade-*_MiSTer` (×~171) | One repo per FPGA core. Each ships its "main" `.mra`(s) in `releases/` (sometimes also mirrored in `mra/`). The `<setname>`/`<parent>`/`<mameversion>`/`<rom zip=...>`/`<part crc=...>` fields are what MAME updates break. | [MiSTer-devel](https://github.com/MiSTer-devel) |
| [`MRA-Alternatives_MiSTer`](https://github.com/MiSTer-devel/MRA-Alternatives_MiSTer) | Region/revision/clone `.mra` variants that ride on an *existing* core's `.rbf` (via the `<rbf>` tag) but aren't published in that core's own repo. Organized as `_alternatives/<Game Family>/*.mra`. **This is easy to forget** — a MAME rename/reparent can hit a set that only exists here, never in the owning `Arcade-*_MiSTer` repo. | MiSTer-devel |
| [`ArcadeDatabase_MiSTer`](https://github.com/MiSTer-devel/ArcadeDatabase_MiSTer) | A hand-maintained `ArcadeDatabase.csv` covering naming/region/manufacturer/`parent_title` metadata across *all* arcade content. Useful as a second opinion on parent/clone relationships, but it's display metadata, not romset truth — it can itself be stale relative to a MAME release (it was, for Sailor Moon, as of this writing — see `reports/`). | MiSTer-devel |

A known gap: a handful of `Arcade-*_MiSTer` repos (recognizable by a `jt*`
subfolder, a `vendoring/` folder, or a `jtframe_*.sdc` file — e.g.
`Arcade-1942_MiSTer`, `Arcade-1943_MiSTer`, `Arcade-Deco156Simple_Mister`) are
[Jotego](https://github.com/jotego)-framework cores. They **do not publish `.mra`
files in the MiSTer-devel repo at all** — Jotego distributes those separately
through his own `jotego/jtbin` repo. This audit pack does not currently cover
that repo; treat any Jotego-pattern core as out of scope until `jotego/jtbin` is
added as a fourth source. Don't treat "no `.mra` found" for one of these as a
bug in your snapshot — verify the jt*/vendoring marker before concluding that.

`Arcade-CV1k` exists as a name reservation but is an empty repo (no commits) —
skip it.

## When actually editing a `.mra`: only `releases/` counts

For `Arcade-*_MiSTer` core repos, **only edit `.mra` files that live under
`releases/`** (its own subfolders count too — `releases/_alternatives/`,
`releases/Alternative Sets/`, `releases/Unsupported/`, etc. are still
`releases/`, fair game). **Don't edit anything outside `releases/`** — most
commonly a `docs/` folder — even though `.mra` files sometimes live there
too and a check script will happily find and flag them. Two real examples
from doing this: `Arcade-IremM72_MiSTer/docs/irem_m84_mra/Cosmic Cop
(World).mra` has no `releases/` counterpart at all — it was never promoted
to a live release, editing it doesn't ship anything to end users.
`Arcade-Sonson_MiSTer/docs/MiST/Capcom SonSon/meta/SonSon.mra` looks like a
duplicate of the real `releases/SonSon.mra` but isn't — that whole `docs/MiST/`
tree is leftover reference material from **MiST**, a different, predecessor
FPGA platform, not MiSTer at all.

It's fine to *read*/audit these paths — a full sweep should still find and
report them, since they're real signal about repo hygiene — just don't spend
fix effort on them. This restriction is specific to the core repos; it
doesn't apply to `MRA-Alternatives_MiSTer`, whose entire structure lives
under `_alternatives/` by design and is the normal, expected place to edit
there.

## Refreshing the repo list (do this first, every run)

The list of `Arcade-*_MiSTer` repos changes as cores are added. Don't trust a
stored list. Refresh it live:

```
GET https://api.github.com/orgs/MiSTer-devel/repos?per_page=100&page=1..N
```
filter to names starting with `Arcade-` (plus the one outlier,
`Arcade_SkySmasher_MiSTer`, which uses an underscore). **Unauthenticated
`api.github.com` calls are capped at 60/hour** — that's enough for the ~4 pages
this takes, but don't also use it to list each repo's file tree (see below).

As of 2026-08-06 this was **172 repos** (171 `Arcade-` + 1 `Arcade_`),
**169 of which** currently publish at least one `.mra` in-repo (see the gap
note above for the other 3).

## Pulling `.mra` contents without hitting API limits

Don't use `api.github.com/repos/.../contents/releases` per repo to discover
filenames — that's 170+ calls against a 60/hour budget and will fail partway
through. Instead, shallow/sparse `git clone` each repo over the git smart-HTTP
protocol, which isn't subject to the REST rate limit at all:

```bash
git clone --filter=blob:none --no-checkout --depth 1 -q \
  https://github.com/MiSTer-devel/<repo>.git dest
cd dest
git sparse-checkout init --no-cone      # NOTE: no -q flag — this git build rejects it and the checkout silently no-ops if you chain with && past the error
git sparse-checkout set '*.mra' 'releases/*.mra' 'mra/*.mra'
git checkout                             # also no -q
```

This pulls only the small text `.mra` files, not the multi-megabyte `.rbf`
bitstreams sitting alongside them in `releases/`. A full pass over all 172
repos this way took well under 10 minutes and a few MB of transfer. For
`MRA-Alternatives_MiSTer`, a full (non-sparse) clone is fine — it's all text,
no binaries, ~3 MB total for all ~810 `.mra` files.

Extract the fields you need with a simple regex over each `.mra` (they're not
always well-formed enough for a strict XML parser — attribute-only tags like
`<switches default="...">` are common, and a couple of files have stray
encoding issues): `<name>`, `<setname>`, `<parent>`, `<mameversion>`, `<rbf>`,
`<year>`, `<manufacturer>`.

## Incremental sweeps: only checking `.mra`s changed since last time

For a full-repo sweep (`mra_rom_check.sh` across every core repo, or the full
cross-reference against current MAME) — as opposed to the version-by-version
changelog procedure, which is already incremental by nature — don't re-pull
and re-check all ~1,850 files across all 182 repos every time. Most repos
haven't changed since the last sweep at all. **`data/mra-rom-check-state.tsv`**
tracks the commit SHA each repo was at when its `.mra`s were last actually
examined (one row per repo: `repo`, `last_checked_sha`, `last_checked_date`).
Use it to skip untouched repos entirely and, for touched ones, check only the
files that actually changed:

1. **Cheap SHA check, every repo, every run:**
   ```bash
   git ls-remote https://github.com/MiSTer-devel/<repo>.git HEAD
   ```
   No clone needed — this is a single lightweight git-protocol round-trip per
   repo, not a REST call, so it doesn't touch the 60/hour API budget either.
   Compare the returned SHA against the recorded `last_checked_sha`. **If it
   matches, skip the repo completely** — nothing under `releases/` can have
   changed if the repo's HEAD hasn't moved. In practice most repos match on
   any given day; this is what makes the sweep cheap to re-run often instead
   of treating it as an occasional heavy operation.

2. **For repos where the SHA differs** (or that have no row yet — a brand-new
   repo, or the first time this mechanism is used): clone with full history
   but no blob content, same technique as MAME's "rename archaeology"
   (`--filter=blob:none`, no `--depth` limit — depth-limiting would make an
   old `last_checked_sha` unreachable to diff against):
   ```bash
   git clone --filter=blob:none --no-checkout https://github.com/MiSTer-devel/<repo>.git dest
   cd dest
   git diff --name-status <last_checked_sha> <new_sha> -- 'releases/*.mra' 'releases/**/*.mra'
   ```
   This is a tree-only diff (no blob fetch), so it's fast regardless of how
   much history sits between the two SHAs. It returns exactly the `.mra`
   paths that were **A**dded or **M**odified under `releases/` — deleted
   files (`D`) obviously don't need checking. If a repo has no prior row,
   there's no baseline to diff from — check its entire current `releases/`
   tree instead, same as a first-time repo.
3. **Materialize and check only those paths:** `git sparse-checkout set` to
   just the changed paths (not the whole repo), `git checkout <new_sha>`,
   then run `mra_rom_check.sh -m <dir> -r` (or `-f` per file) over just that
   small set. For the full cross-reference check (setname/CRC verification
   against MAME source, not just the structural script), the same changed-file
   list is the thing to actually re-verify — no need to re-check files that
   didn't change.
4. **Update the state file** after checking: set `last_checked_sha` to
   `<new_sha>` and `last_checked_date` to today, for every repo actually
   checked (including ones that came back with zero relevant changes — HEAD
   moved but nothing under `releases/` did, e.g. an `.rbf`/`.sv` update).
   Leave rows untouched for repos that were skipped at step 1 (their recorded
   SHA is still accurate — they haven't moved).

This means "check everything since the last time" degrades gracefully: a
same-day re-run costs ~182 `ls-remote` calls and touches nothing else; a
sweep after a long gap does real work but only for the repos that actually
changed, and only re-examines the files that actually changed within them.
The state file was seeded from the 2026-08-23 full sweep (`reports/2026-08-23-mra-rom-check-resweep.md`)
— every repo's row reflects the SHA examined that day, whether or not
findings from that sweep have since been fixed/pushed upstream. Don't confuse
"checked" with "clean" — a repo can be recorded as checked at a SHA that's
known (via mame.md's Open Findings) to still have unresolved issues.

## Snapshot data (as of 2026-08-06)

Three files in [`data/`](data/), generated with the method above:

- **`data/mister-devel-repo-index.tsv`** — one row per `Arcade-*_MiSTer` repo:
  set count, the `<rbf>` value(s) it uses (its MiSTer-side driver-family
  label), top manufacturer, year range, URL. Start here to find which repo
  owns a given game.
- **`data/mister-devel-cores.tsv`** — one row per `.mra` file across all core
  repos (~1040 rows): repo, filename, name, setname, parent, mameversion,
  rbf, year, manufacturer. This is the thing that goes stale — treat it as a
  dated baseline, not live truth, and re-pull before trusting it for anything
  more than a few weeks old.
- **`data/mra-alternatives.tsv`** — same shape, for the ~810 files in
  `MRA-Alternatives_MiSTer/_alternatives/`, plus which game-family folder each
  came from.
- **`data/mame-current-sets.tsv`** — the MAME-side counterpart: every
  `setname`/`parent`/`year`/`source_file` currently defined anywhere in MAME
  (~31.8k rows), pulled the same rate-limit-free way. See mame.md's "Full
  cross-reference against current MAME" section for the method and for two
  false-positive traps found doing this (MiSTer's inconsistent `<parent>` tag
  usage, and HBMAME-sourced content in `MRA-Alternatives_MiSTer` that was
  never part of mainline MAME to begin with).

Regenerate all four (methods above) as the first step of every audit run, and
update the date in this section when you do.

## `<rbf>` is not a MAME driver filename

The `<rbf>` tag in an `.mra` is MiSTer's own short label for the core/hardware
family (e.g. `CaveBanpresto`, `IremM92`, `NightSlashers`, `BubSys`, `BubSysROM`,
`atarisys1`). It's a useful grouping key across `.mra` files that share
hardware, but it does **not** correspond 1:1 to a MAME source file path, and
MAME renaming a `.cpp` file (this happens most releases) does not by itself
mean anything needs to change on the MiSTer side — only a setname, parent, or
ROM content change does. Don't flag driver-file-only renames as findings;
verify against the actual `ROM_START`/`GAME()` macros (see mame.md) before
concluding anything is broken.

## Verifying a specific game

1. Find its repo: grep `data/mister-devel-repo-index.tsv` for the game name,
   or grep `data/mister-devel-cores.tsv` / `data/mra-alternatives.tsv` for its
   setname.
2. Pull the live `.mra` (git sparse-clone as above, or
   `raw.githubusercontent.com/MiSTer-devel/<repo>/<branch>/releases/<file>.mra`
   once you know the exact filename — raw fetches aren't rate-limited).
3. Compare its `<setname>`/`<parent>`/`<rom zip=...>`/`<part crc=...>` against
   MAME's current source for that driver (see mame.md's verification
   technique) — not against the stored TSV snapshot, which may already be
   stale by the time you're reading it.
