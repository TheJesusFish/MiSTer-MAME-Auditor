# MiSTer MAME Auditor

A drop-in context pack — no code — for checking whether a MAME release broke
any [MiSTer-devel](https://github.com/MiSTer-devel) arcade core's ROM
definitions. MAME renames/reparents/restructures romsets most releases; MiSTer's
`Arcade-*_MiSTer` cores (and the separate `MRA-Alternatives_MiSTer` repo of
clone/region variants) reference those romsets by exact setname/CRC in their
`.mra` files, and silently drift out of sync when MAME changes underneath them.

## Use it

Point any agent (Claude Code, Codex, etc.) at this repo and say something like:

> Check the latest MAME release against mister-devel and tell me what needs
> updating. Follow prompt.md.

The agent should read `prompt.md` first — it explains the procedure and points
to everything else.

## What's in here

| File | Purpose |
|---|---|
| [`prompt.md`](prompt.md) | The task: what to do, in order. Start here. |
| [`mister-devel.md`](mister-devel.md) | The MiSTer side — the three repos involved, how to refresh their contents without hitting GitHub's API rate limit, and known quirks (Jotego cores, `<rbf>` vs. MAME driver paths). |
| [`mame.md`](mame.md) | The MAME side — where the changelog lives, what it does and doesn't tell you structurally, how to verify a lead against live driver source, and a **run history log** of what's already been audited. |
| [`data/`](data/) | Dated snapshots (TSV) of MiSTer's current `.mra` contents, and of MAME's current setname/parent catalog. Regenerated at the start of each run. Treat as a baseline, not live truth. |
| [`reports/`](reports/) | One dated findings report per audit run — both the regular changelog-driven kind and the occasional full cross-reference kind (see mame.md). |

## Why it's structured this way

The two failure modes this is built around:

1. **MAME's changelog isn't structured data.** Older MAME versions had a clean
   "renamed machines" section; current releases don't. Renames and reparenting
   are described in prose scattered through a few-thousand-line changelog, so
   this has to be *read*, not diffed — `mame.md` sets that expectation and
   gives keyword leads, but the real verification step is checking the actual
   MAME driver source (also explained there).
2. **There isn't one MiSTer-side source of truth.** A game's romset can be
   defined in its own `Arcade-*_MiSTer` repo, only in `MRA-Alternatives_MiSTer`,
   or (inconsistently) in both — and they can fall out of sync with each other,
   not just with MAME. `mister-devel.md` treats both as first-class, plus notes
   a related repo (`ArcadeDatabase_MiSTer`) that carries its own naming metadata
   and can itself go stale (it did, for the 0.289 Sailor Moon reparent).

## Status

Seeded 2026-08-06 with two audits:

- A changelog-driven pass against MAME 0.289 —
  [`reports/2026-08-06-mame-0289-audit.md`](reports/2026-08-06-mame-0289-audit.md).
- A full cross-reference of every mister-devel/`MRA-Alternatives_MiSTer`
  setname against current MAME source (not tied to one release, catches older
  drift the changelog method can't) —
  [`reports/2026-08-06-full-mame-crossref-audit.md`](reports/2026-08-06-full-mame-crossref-audit.md).

The full cross-reference pass left 10 setnames with no obvious current-MAME
match, followed up same day with git history archaeology (`git log -S` on a
local MAME history clone — see mame.md's "Rename archaeology" section for the
technique) rather than leaving them as open questions: **7 turned out to be
confirmed MAME renames from 2020–2023**, unrelated to 0.289, and **3 came up
empty across the driver's full tracked history** (presumed never-official,
not pursued further). Fan hacks, patches, and content MAME doesn't emulate
that turned up along the way were checked once and deliberately dropped, not
tracked as findings.

**6 of the resulting 13 confirmed renames have since been fixed**, scoped to
`MRA-Alternatives_MiSTer` clone `.mra`s plus `Arcade-SpaceFirebird_MiSTer`
(the one main-repo case that's an actual clone of its own flagship set, not
bonus content) — `pengo2/4/5`, `joustwr`, `bagmans2`, `sinistar1`,
`SpaceDemon`, plus deduping a stale/correct `gauntletr8`/`gauntletgr8`
duplicate pair. **Committed locally in two repo clones, not yet pushed or
opened as PRs upstream.** The other 7 (including `twinbeeb`, which already
has an upstream PR) are deliberately out of scope for this batch — see
mame.md's "Open findings" for the live status of everything, fixed and not.
Next changelog-driven run picks up at **0.290**; `mame.md`'s run history has
the full detail.
