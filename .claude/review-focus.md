This project is an HD remaster toolkit for The Space Bar (1997), a game that runs on ScummVM's BAGEL engine. Two kinds of change come through here: Python tools in spacebar_tools/ (with tests in tests/), and C++ engine changes for ScummVM's engines/bagel/ that ship as .patch files. Judge each by the rules for its kind. Anything that would make the game crash, load a corrupted asset, or silently produce wrong output for thousands of files is at least MAJOR.

## Python tools

- Python 3.14. ruff must stay clean. Every behaviour change needs a test under tests/, and existing tests must not be weakened to pass.
- WLD writer round-trip: parsing then writing must reproduce non-coordinate content byte for byte. Scaling may touch only coordinates — rects `[x1,y1,x2,y2]`, hotspots `@[hotX,hotY]`, positions `[x,y]` — and never TURNCOUNT values, VAR assignments, timer thresholds, PANIM frame indices, percentages or cursor definitions. Flag any regex or tokenizer that could match a non-coordinate number.
- Idempotency: batch scalers (WLD, BIN, BMP, SMK→AVI) write into an override directory that mirrors the game tree. Running twice must not double-scale. Flag any runner that cannot tell scaled output from source, or that reads from the directory it writes to.
- BIN files are int32 little-endian x,y pairs. Scaled values must stay in range, and rounding must match the WLD scaler exactly or sprites drift from their hotspots.
- BMP output must be 8-bit CLUT8 using the source file's exact palette (the game uses USESHAREDPAL — never re-derive a palette). Palette index 1 (RGB 255,7,162) is the global chroma key and must be excluded from dithering and quantization. 24-bit output or a re-derived palette is a BLOCKER.
- Video: SMK→AVI. The engine's decoder factory is extension-driven, so output must be `.avi`; the BIN companion is scaled in the same pass at the same factor.
- File handling at scale: ~5,900 assets, ~1.8 GB. Flag loading whole trees into memory, and silent skips on error — a skipped asset is a missing texture in the game. Watch Windows paths: backslashes, case-insensitive matching, the `$SBARDIR\` prefix, long paths.
- Never modify the game's original files. `sbar-timer patch` is the one tool that writes into the game directory, and it must back up first and stay reversible via `sbar-timer restore`. Any other in-place write to source assets is a BLOCKER.
- Timer: any change that freezes TURNCOUNT or stops it incrementing is a BLOCKER — it breaks bar progression. Only the game-over checks (`TURNCOUNT >= 2250 ...`) and the engine hard cap may change.

## Engine patches (C++, ScummVM engines/bagel/)

- Changes ship as `git apply`-able patches against a clean ScummVM checkout. A diff that mixes the AVI/override work with the resolution work (two separate patch files) or would not apply cleanly is MAJOR.
- Only engines/bagel/spacebar/, the engines/bagel/ entry points already patched, and the three shared files the patches already touch (video/avi_decoder.cpp, video/avi_decoder.h, image/codecs/bmp_raw.cpp). Changes to those three affect every engine in ScummVM: flag anything that alters behaviour for non-BAGEL callers. engines/bagel/hodjnpodj/ is a different game on the same engine: any change there is a BLOCKER unless the task says otherwise. Any other file outside engines/bagel/ is MAJOR unless the task names it.
- CBofBitmap is CLUT8-only. Any path that hands it 24-bit data is a BLOCKER.
- Resolution constants: DEF_WIDTH/DEF_HEIGHT are inclusive maxima (639/479 → 1279/959). The main window is DEF_WIDTH+1 × DEF_HEIGHT+1 = 1280×960, not PAN_AREA_WIDTH × PAN_AREA_HEIGHT. Flag any new hardcoded 640/480/320/240/160 and any off-by-one on the +1.
- Panoramic math is ratio-based through MAX_DIV_VIEW (≈4.267) and needs no change for 2×. Flag anything that hardcodes a viewport width instead.
- Warp correction must stay on the cosine table (Correction=1 in scummvm.ini); the precomputed lookup table (Correction=2) overflows with 1024-tall panoramas. Flag anything that reintroduces or depends on the LUT.
- AVI fallback: the extension swap exists at three points (spacebar.cpp, movie_object.cpp, character_object.cpp). A new video-loading path that bypasses the same swap is MAJOR.
- Override priority: extrapath at SearchMan priority 1, game data at 0. Anything that registers an archive at priority ≥1 or reorders initializePath silently breaks overrides.
- TURNCOUNT hard cap at var.cpp:304 (`== 2250` → skip increment). Raising it must be paired with the WLD threshold change; lowering it is a BLOCKER.
- Match the surrounding ScummVM code style.
- Rendering, panning, click alignment and audio cannot be verified from a diff. Label such findings UNVERIFIED and say exactly what to check on screen.
