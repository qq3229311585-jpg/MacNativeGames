# Handoff: colour-pass holes / ghosting while the camera moves (Bayonetta 3)

Status (2026-11-25): **open, needs someone with the game.** The probe below is written but has never been run in
the game, and `nvn_mtl.m` with it has not been compiled on macOS. Whoever picks this up: please build first, then run.

## The symptom

In play (ship deck, prologue), moving the camera leaves trails and blocks, and moving things look translucent.
With a still camera (cutscenes) the picture is right. History and earlier measurements: `games/by3/CHANGELOG.md`,
entries of 2026-10-19.

## What is already known

- Frame structure: pass 830488 is a depth-only pre-pass (563 draws), and pass 830508 is the colour pass into
  ser 41 with `LEQUAL` on that depth. Ser 41 is loaded, not cleared, so a pixel that no draw writes keeps the previous frame.
- `DM_EXP_CLEAR_SER=41` (magenta for uncovered pixels): with the camera moving, large regions stay magenta.
- `DM_EXP_NODEPTH_SER=41` (no depth test in the colour pass) makes every hole disappear, so the colour-pass draws
  arrive and fail the depth test. The two passes place the same surfaces at different depths, and only while the
  camera moves.
- Holes remain with each of these: exact fences, `DM_PRESERVE_INVARIANCE=1` (the default in by3), `TOTK_UB_RENAME=0`,
  `TOTK_UBO_SNAPSHOT=0`, and `DM_EXP_SUBMIT_WAIT=1`.
- Ruled out from the code (2026-11-25):
  - Depth store dropped at pass end: the TBDR store lookahead is off by default (`DM_NVN_TBDR` unset), and
    `sd_finish` then always uses `MTLStoreActionStore`.
  - Old shader binaries without `preserveInvariance`: by3 has no prebuilt `shader-libs`. Its translated shaders
    are compiled at launch with `totk_precise_opts()`, which sets the flag.

## Working hypothesis

One of the two passes reads another frame's camera. A typical cause is a binding or offset mixed up between the
pre-pass program and the colour-pass program, for example the previous-frame view-projection that the colour pass
keeps for motion vectors. When the camera is still, the current and previous matrices are equal, which would
explain why the bug appears only while the camera moves.

## The probe: `DM_EXP_CAMCMP`

Code: `games/by3/third_party/d2r_gfx/gfx/nvn/nvn_mtl.m`. Search for `camcmp`.

```
DM_EXP_CAMCMP=41            # serial of the colour-pass target
DM_EXP_CAMCMP_FRAMES=8      # optional: frames to report (default 8)
```

Use the same scripted deck run as in the CHANGELOG: enter the deck fight, then hold W and turn the camera with the
arrow keys. For each frame, the probe hashes the vertex-stage uniform bytes as they are bound, which is what the
GPU reads: the submit-time snapshot, a renamed inline update, or memory. It does this for the depth pre-pass
(depth write on, no colour) and for the colour pass into ser 41, then prints one `[camcmp]` report per frame:

- `(a)`: guest uniform ranges that both passes bind with different bytes.
- `(b)`: draws found in both passes (same index buffer, counts and vertex buffers), and which `cbN` slots differ,
  either at the same address (`同址`) or at a different address (`异址`).
- For each difference, the first differing 16-byte rows are printed as floats.
- Source letters: `m` is guest memory, `c` is a copy (snapshot or renamed inline update), and `l` is live bytes.

While the probe runs, argument-buffer reuse is turned off so that every draw's bindings are seen.

## How to read the result

- **Rows differ only while the camera moves, and look like a 4x4 matrix step**: one pass reads another frame's
  camera. The slot and the source letter show where the stale copy comes from. Compare the copy path (`c`,
  snapshot / `upd_shadow`) with memory (`m`), and check which pass's value matches this frame.
- **(a) reports no differences, but (b) reports `异址`**: the two programs bind different buffers for the camera.
  Check the translated shaders' cbuf slots and offsets (the `K`/`B` lines from `DM_NVN_PASSDUMP` +
  `DM_NVN_CBDUMP` give the full bytes).
- **No differences at all**: the uniforms are not the cause. The next suspects are the vertex streams (skinning or
  positions in storage buffers, which the probe does not hash) and the shaders' own arithmetic. For the
  arithmetic, compare the two vertex shaders' MSL for the position calculation.

## Files

- Probe: `nvn_mtl.m`. The code block starts at `#define CC_ROWS`. Hooks: `camcmp_note_cbuf` in `encode_args`
  (CBUF branch), and `camcmp_draw` after `encode_args` in `draw_inner`.
- Measurement switches used before: `DM_EXP_CLEAR_SER`, `DM_EXP_NODEPTH_SER`, `DM_EXP_SUBMIT_WAIT`,
  `gpucapture <path>.gputrace` (input file command; needs `MTL_CAPTURE_ENABLED=1`).
