# CLAUDE.md — ScummVM checkout for the Space Bar HD remaster

<!-- Paths: the tools repo is the sibling folder "../Spacebar HD" (note the space - quote it in commands). -->

This is a clone of upstream ScummVM used for one thing: developing the BAGEL engine changes
that the Space Bar HD remaster needs, at `HD_SCALE` = 3 (1920×1440). The plan, the Python
tools and the shipped `.patch` file live in the companion repo `../Spacebar HD` (the Python
project is in its `spacebar-hd-tools/` subfolder). Read its root `CLAUDE.md` and
`SpaceBar_HD_Project_Plan.md` (ScummVM patch, Technical Constraints) before any engine work
here. The open engine findings are in its `docs/engine-review-2026-09-26.md` (plan stories
E1–E3).

## Branch model — keep the working tree clean

- All Space Bar changes live on the branch `spacebar-hd`. `master` tracks `upstream`
  (scummvm/scummvm) untouched. The branch is pushed only to Matt's own remote (`origin`:
  his fork or private mirror). Never open a pull request upstream from it.
- This is a blobless partial clone (`--filter=blob:none`): file contents for older commits
  are fetched from `upstream` on demand, so `git log -p`, `git blame`, `git show <old>:<file>`
  and rebases may hit the network. That is normal, not an error.
- Build leftovers in the tree (`*.dll`, `scummvm.exe`, `config.*`) are kept out of git by
  `.git/info/exclude`. Never `git add` them; if new ones appear, add a pattern there.
- The base is upstream `7cf428776` (= local `master`). `upstream/master` has moved on
  (`12fa21690` on 2026-09-26); that's expected, not drift to fix.
- Taking upstream changes is Matt's call, not a routine step: `git fetch upstream`, then
  rebase `spacebar-hd` onto `upstream/master` (rebase, not merge, so the patch stays
  regenerable), then `git push --force-with-lease`, regenerate the patch and record the new
  base commit in the plan.
- `git worktree add ../scummvm-clean 7cf428776` gives a clean tree of the base for
  `git apply --check` and side-by-side builds without a second clone. Use the base, not
  `upstream/master`.
- Start every task from a clean tree (`git status`), so the reviewer sees only the new change.
  With the engine work committed on the branch, `git diff HEAD` is the task's diff, not the
  whole remaster.
- The engine change ships as **one** patch, `bagel-spacebar-hd.patch` in the tools repo root,
  generated from the branch and never hand-edited (plan story 2A.9, after E1–E3):
  `git diff 7cf428776 spacebar-hd -- engines video image > "../Spacebar HD/bagel-spacebar-hd.patch"`
  (`video/` and `image/` because the patch touches `video/avi_decoder.cpp`,
  `video/avi_decoder.h` and `image/codecs/bmp_raw.cpp`; this also keeps `CLAUDE.md`,
  `.claude/` and `.gitignore` out). One commit per task on the branch is fine; there is no
  remaster/resolution split any more. The three older patch files in the tools repo are stale
  and retire once the new one checks out. Regenerating the patch is an orchestrator task; show
  Matt the `git diff --stat` first.

## What may change here

- `engines/bagel/spacebar/**` and the `engines/bagel/` entry points the patches already touch.
- The three shared files the patches already touch: `video/avi_decoder.cpp`,
  `video/avi_decoder.h`, `image/codecs/bmp_raw.cpp`. Changes there affect every engine in
  ScummVM, so keep them minimal and behaviour-preserving for other games.
- Nothing under `engines/bagel/hodjnpodj/` (a different game on the same engine).
- Nothing else outside `engines/bagel/` without asking Matt first.
- Match the surrounding ScummVM code style.

## Build and test

- Builds run in MSYS2 MinGW64, not Git Bash — see `docs/scummvm/BUILD_GUIDE.md` in the tools
  repo (two `docs/` folders exist there until reconciliation; use whichever has the guide).
  Until a Git Bash-driven build has been proven to work, rebuilding and launching the game are
  Matt's job: end every engine task with a checklist — what to build, which area to load, what
  to look at on screen (the BAR area at 1920×1440 is the integration test, Story 2A.6). The
  `scummvm.exe` in the tree dates from 2026-03-14 and is a 2× build.
- There are no unit tests for the engine changes. `git apply --check` of a regenerated patch
  against a clean worktree of `7cf428776` is the one mechanical check available; use it.
- Save games are resolution-specific: test from a new game.

## Facts that bite (details in the tools repo's CLAUDE.md and the plan)

- `HD_SCALE` (`pan_window.h`) is the one knob, set to 3 by E3. Every size and rect derives
  from it or from `DEF_WIDTH`/`DEF_HEIGHT`; no pre-scaled literals (1280, 960, 1920, 1440 …).
  The branch still holds 2× literals until E3 lands.
- `CBofBitmap` is CLUT8-only; 24-bit data or a palette-less AVI corrupts or crashes. Only the
  four INTRO videos are bgr24, and they play through `CBofMovie`.
- `CBofRect` is inclusive: far edges scale as `HD_SCALE·(r+1) − 1`, points as `HD_SCALE·x`.
- Main window is DEF_WIDTH+1 × DEF_HEIGHT+1 (1920×1440 at `HD_SCALE` 3), not
  PAN_AREA_WIDTH × PAN_AREA_HEIGHT.
- Panoramic math is ratio-based (MAX_DIV_VIEW ≈ 4.267): no hardcoded viewport or panorama
  sizes. `rotateTo` (`pan_window.cpp:720-741`) still hardcodes a 2048-wide panorama (E1).
- Warp correction: cosine (Correction=1) is the only mode at HD. The strip table
  `STRIP_POINTS[153][120]` (`paint_table.h:36`, 1× data from `paint_table.txt`) overflows at
  any `HD_SCALE` > 1, and it is still reachable: `DEFAULT_CORRECTION` is 2
  (`master_win.cpp:1720`), the panorama constructor starts on it (`pan_bitmap.cpp:99`), and
  the Options slider can select it (`opt_window.cpp:279`). E3 closes this.
- AVI fallback: extension swap before every video load (spacebar.cpp, movie_object.cpp,
  character_object.cpp; `EVGAMWIN.SMK` and `OVERRIDE.SMK` still bypass it, E1). Unconverted
  SMKs must keep working, seeking included: Smacker needs `forceSeekToFrame` (E1).
- Override: extrapath at SearchMan priority 1, game data at 0 (`spacebar.cpp:initializePath`).
- TURNCOUNT hard cap: `var.cpp:305`, `== 2250` skips the increment. Never lower it; raise it only together with the WLD threshold.

## Seats

Same seats and loop as the tools repo: you orchestrate and do not write implementation code;
`codex-coder` implements from a task spec (it has no context but the spec, and no network);
`gemini-reviewer` reviews every change before commit and reads `.claude/review-focus.md`
automatically; consult the advisor before touching any engine constant or load path;
`fable-escalation` only after two failed coder attempts or an unknown root cause. Show Matt a
summary and wait for his go-ahead before every commit. Task specs use the template in the
tools repo's CLAUDE.md, with "Verify" replaced by: `git diff --stat`, then the on-screen
checklist for Matt.
