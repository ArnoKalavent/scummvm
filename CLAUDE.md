# CLAUDE.md — ScummVM checkout for the Space Bar HD remaster

<!-- Paths: the tools repo is the sibling folder "../Spacebar HD" (note the space - quote it in commands). -->

This is a clone of upstream ScummVM used for one thing: developing the BAGEL engine changes
that the Space Bar HD remaster needs. The plan, the Python tools and the shipped `.patch`
files live in the companion repo `../Spacebar HD` (the Python project is in its
`spacebar-hd-tools/` subfolder). Read its root `CLAUDE.md` and
`SpaceBar_HD_Project_Plan.md` (Technical Constraints, ScummVM Patches Summary) before any
engine work here.

## Branch model — keep the working tree clean

- All Space Bar changes live on the branch `spacebar-hd`. `master` tracks `upstream`
  (scummvm/scummvm) untouched. The branch is pushed only to Matt's own remote (`origin`:
  his fork or private mirror). Never open a pull request upstream from it.
- This is a blobless partial clone (`--filter=blob:none`): file contents for older commits
  are fetched from `upstream` on demand, so `git log -p`, `git blame`, `git show <old>:<file>`
  and rebases may hit the network. That is normal, not an error.
- Build leftovers in the tree (`*.dll`, `scummvm.exe`, `config.*`) are kept out of git by
  `.git/info/exclude`. Never `git add` them; if new ones appear, add a pattern there.
- Taking upstream changes is Matt's call, not a routine step: `git fetch upstream`, then
  rebase `spacebar-hd` onto `upstream/master` (rebase, not merge, so the patches stay
  regenerable), then `git push --force-with-lease`, regenerate the patch files and record
  the new base commit in the plan.
- `git worktree add ../scummvm-clean upstream/master` gives a clean upstream tree for
  side-by-side builds without a second clone.
- Start every task from a clean tree (`git status`), so the reviewer sees only the new change.
  With the patches committed on the branch, `git diff HEAD` is the task's diff, not the whole
  remaster.
- The `.patch` files in the tools repo are generated from the branch, never hand-edited.
  Keep the remaster (AVI decoder, override priority, audio fix) and resolution (viewport
  constants) changes in separate commits so each patch can be regenerated on its own:
  `git diff <base> <commit> -- . ':!CLAUDE.md' ':!.claude' ':!.gitignore' > "../Spacebar HD/<name>.patch"`
  (exclude the config files rather than restricting to `engines/bagel`: the patches also
  touch `video/avi_decoder.cpp`, `video/avi_decoder.h` and `image/codecs/bmp_raw.cpp`).
  Regenerating the patches is an orchestrator task; show Matt the `git diff --stat` first.

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
  to look at on screen (the BAR area at 1280×960 is the integration test, Story 2A.6).
- There are no unit tests for the engine changes. `git apply --check` of a regenerated patch
  against a clean `master` worktree is the one mechanical check available; use it.

## Facts that bite (details in the tools repo's CLAUDE.md and the plan)

- `CBofBitmap` is CLUT8-only; 24-bit data corrupts.
- Main window is DEF_WIDTH+1 × DEF_HEIGHT+1 (1280×960), not PAN_AREA_WIDTH × PAN_AREA_HEIGHT.
- Panoramic math is ratio-based (MAX_DIV_VIEW ≈ 4.267): no hardcoded viewport widths.
- Warp correction: cosine table (Correction=1); the LUT (Correction=2) overflows at 1024-tall panoramas.
- AVI fallback: extension swap at three points — spacebar.cpp, movie_object.cpp, character_object.cpp.
- Override: extrapath at SearchMan priority 1, game data at 0 (`spacebar.cpp:initializePath`).
- TURNCOUNT hard cap: `var.cpp:304`, `== 2250` skips the increment. Never lower it; raise it only together with the WLD threshold.

## Seats

Same seats and loop as the tools repo: you orchestrate and do not write implementation code;
`codex-coder` implements from a task spec (it has no context but the spec, and no network);
`gemini-reviewer` reviews every change before commit and reads `.claude/review-focus.md`
automatically; consult the advisor before touching any engine constant or load path;
`fable-escalation` only after two failed coder attempts or an unknown root cause. Show Matt a
summary and wait for his go-ahead before every commit. Task specs use the template in the
tools repo's CLAUDE.md, with "Verify" replaced by: `git diff --stat`, then the on-screen
checklist for Matt.
