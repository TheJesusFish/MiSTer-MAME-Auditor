# Task: audit mister-devel arcade cores against the latest MAME release

You're checking whether recent MAME changes (romset renames, reparenting,
clone/parent swaps, driver splits that move ROM content) have broken any
`.mra` definitions in the [MiSTer-devel](https://github.com/MiSTer-devel)
arcade cores, or in the separate `MRA-Alternatives_MiSTer` repo of clone/region
variants. MiSTer arcade cores are hardware reproductions that load MAME's
romsets by filename/CRC; when MAME restructures a romset, MiSTer's `.mra`
needs a matching update or the core stops finding its ROMs.

Read **[mister-devel.md](mister-devel.md)** and **[mame.md](mame.md)** in full
before doing anything else — they contain the methodology (how to pull data
without hitting GitHub's rate limit, how to tell a real romset break from
changelog noise) and the state from the last run. Don't reinvent either; both
files exist because the naive approaches (per-repo `api.github.com` listing
calls, trusting the changelog's prose without checking source) don't work or
are slow enough to fail partway through.

## Procedure

1. **Refresh the MiSTer-side snapshot.** Follow mister-devel.md's method to
   re-pull the current list of `Arcade-*_MiSTer` repos and re-extract
   `.mra` fields into `data/*.tsv`. Do this even if the data looks recent —
   it's cheap (a few minutes) and the whole point is not trusting a stale
   snapshot. Update the "as of" date in mister-devel.md when done.

2. **Check standing open findings first.** mame.md has an "Open findings"
   section — confirmed and to-review issues from past runs that don't get
   auto-closed just because a changelog got read. For each, pull the live
   `.mra` (mister-devel.md's method) and check whether it's been fixed
   upstream since it was raised. Fold the result into this run's report
   (mark it fixed, or carry it forward) rather than silently re-discovering
   or silently dropping it.

3. **Find where the last run left off.** Read the run history at the bottom
   of mame.md — it names the last MAME version fully audited.

4. **Catch up on the changelog gap.** For every MAME version between the last
   audited one and the current latest (mame.md explains where to find both),
   pull `whatsnew_0XXX.txt` and read it in full — see mame.md for what to
   scan for and how to tell romset-breaking changes from cosmetic ones.

   This changelog method only catches what changed *in the versions you're
   reading* — it won't surface older drift that was already there before this
   pack existed. mame.md also documents a heavier **full cross-reference**
   method (clone current MAME source locally, check every mister-devel
   setname for existence against it) that catches that older drift too, and a
   **`zip=` completeness sweep** (every `.mra`'s zip reference actually
   listing both its parent and its own zip, not just a correct setname) that's
   independent of any rename and won't be caught by either of the other two
   methods. Neither is part of the default per-version procedure — don't run
   them every time — but do run them if explicitly asked for a "full" audit,
   or if it's been many changelog-driven runs since the last one (mame.md's
   run history says when that last was). Same for a `mra_rom_check.sh` sweep
   (structural validity, not a MAME-version question at all). For any of
   these full-repo sweeps, use mister-devel.md's **"Incremental sweeps"**
   method — don't re-pull and re-check all ~1,850 files every time; a cheap
   per-repo SHA check against `data/mra-rom-check-state.tsv` skips whatever
   hasn't changed since it was last actually examined.

5. **Cross-reference.** For each lead, check whether MiSTer covers that game
   (grep `data/mister-devel-repo-index.tsv` and the two `data/*.tsv`
   snapshots by setname or game name) and whether the `.mra` on record
   matches what MAME's current driver source says (mame.md's verification
   technique). Check *both* the owning `Arcade-*_MiSTer` repo and
   `MRA-Alternatives_MiSTer` — a set can live in either, or both, and they
   can be out of sync with each other, not just with MAME.

6. **Report findings.** Write a dated report to `reports/YYYY-MM-DD-mame-0XXX-audit.md`
   (one file per run, covering however many versions it caught up on).
   For each finding:
   - the game, its MiSTer repo/file, and its `MRA-Alternatives_MiSTer` entry
     if any
   - what MAME changed (old → new setname/parent, quoting the changelog line)
   - what MiSTer currently has (quote the relevant `.mra` fields)
   - confidence: confirmed against live MAME source, vs. just the changelog
     prose
   - suggested fix, in enough detail that someone could act on it without
     re-deriving your research (exact new `<setname>`/`<parent>`/`<rom zip>`
     values where you were able to confirm them)

   Separate confirmed breakage from "worth a human look" from "opportunity"
   (new MAME clone/set that doesn't exist in mister-devel yet and could be
   added) — don't flag opportunities as bugs.

   If a setname doesn't exist in current MAME and there's no obvious
   replacement, don't stop at "needs manual review" — mame.md's "Rename
   archaeology" section has a technique for this (`git log -S` on a local
   history clone of MAME) that resolved 7 of 10 such cases in one pass here.
   It's cheap enough to use on the handful of setnames that reach this point.
   If it genuinely finds nothing across the driver's full tracked history,
   that's a real answer too (probably never an official MAME setname) — say
   so and stop, don't leave it as an open "needs review" forever.

7. **Update the log.** Append a summary entry to mame.md's run history (the
   format is already there — follow it) so the next run doesn't redo this
   work. Note explicitly which version to start from next time, and update
   the "Open findings" section — add anything newly discovered, remove
   anything you confirmed is now fixed.

## Ground rules

- **Don't guess at exact CRCs or set relationships from the changelog prose
  alone.** Verify against MAME's actual driver source (mame.md explains how)
  before calling something confirmed. If you can't verify, say so and mark it
  lower confidence rather than asserting it.
- **This is a review aid, not an automated fixer.** Produce findings and
  suggested `.mra` values; don't open PRs against MiSTer-devel repos unless
  explicitly asked to. If you *are* asked to actually apply a fix, read
  mame.md's "Applying a confirmed rename" section first — swapping the
  `<setname>`/`<parent>` tags is not enough on its own (CRCs, the `.mra`
  filename, and the `<rom zip="...">` attribute all need checking too, and
  the filename call in particular depends on the local folder's existing
  convention, not a blanket rule). Leave `<about>` empty — provenance notes
  belong in the commit message and this repo's reports, not in the `.mra`.
- **Any time you're editing an `.mra`'s `zip=` reference — a rename or
  anything else — also check it's complete, not just correctly renamed.**
  mame.md's "Checking `zip=` completeness" section has the convention
  (parent's zip first, clone's own zip second) and how to verify it against
  MAME's *actual* current parent rather than the `.mra`'s own `<parent>` tag
  (the two can legitimately differ). This isn't optional cleanup — a set
  missing its own zip reference can fail to load for users on certain romset
  packaging styles even though the setname itself is correct.
- **Byte-for-byte CRC verification of every existing romset is out of scope**
  for a single run — there are ~1,850 `.mra` files across the ecosystem, no
  MAME binary is available, and most releases only touch a handful of drivers
  in ways that matter. Scope the check to what the changelog actually flags,
  read in full, not a sample of it.
- If something looks like it needs a source-file-only rename tracked (MAME
  renaming a `.cpp`, not a set) — that's almost never a MiSTer-side issue on
  its own; see mister-devel.md's note on `<rbf>` vs. driver paths before
  reporting it.
- **Don't track fan hacks, patches, or content MAME never emulated as open
  findings.** If a setname turns out to be a hack/trainer/patch with no
  official MAME romset behind it (HBMAME content, "(inv)"/invincibility
  patches, "Free Play" patches, ROM patches, etc.) or a game MAME doesn't
  emulate at all (e.g. discrete-logic TTL games with no ROMs) — verify that
  once, mention it in the report so the reasoning is on record, and then drop
  it. Don't carry it forward in mame.md's "Open findings" or re-flag it on a
  future run.
- **In `Arcade-*_MiSTer` core repos, only edit `.mra` files under `releases/`**
  (mister-devel.md's "When actually editing a `.mra`" section has the full
  rule and two real examples of why — a `docs/` folder can hold `.mra` files
  that were never promoted to a live release, or leftover reference material
  from an unrelated project entirely). Fine to audit/report on files outside
  `releases/` if a sweep finds them; don't spend fix effort on them. Doesn't
  apply to `MRA-Alternatives_MiSTer`, which is edited under `_alternatives/`
  by design.
- **Any time a sweep claims to have excluded findings already tracked in
  `MRA-Alternatives_MiSTer`, actually run that comparison and show the
  result — don't just assert it.** A 2026-08-23 sweep claimed "none
  excluded" without the comparison having actually run correctly, and the
  claim went unnoticed until the user pushed back on one repo specifically.
  The fix is cheap and mechanical: sort both setname lists (the sweep's
  failing setnames, and a fresh `MRA-Alternatives_MiSTer` clone's setnames)
  and run `comm -12` between them — a few seconds, and it's verifiable
  rather than asserted. Do this as a named step, every time this filter is
  claimed to have been applied, not just when someone asks.
