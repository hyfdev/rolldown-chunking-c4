# rolldown-chunking-c4

**https://hyfdev.github.io/rolldown-chunking-c4/**

Rolldown's chunk generation modelled twice: what `main` does today, and the
intended layering. Written in [LikeC4](https://likec4.dev).

The `now` model is read from rolldown `main`, commit `6a54a0531`.

## Viewing it locally

```bash
npm install
npm run dev     # http://localhost:5173
```

Editing `src/*.c4` hot-reloads. Click a node for its full text — node boxes
truncate, the details panel does not.

## Two views

| View | Content |
| --- | --- |
| `index` | Today on top, the intended model below, both left to right |
| `proposal_only` | The intended model on its own |

## Colours

| Colour | Marker | Meaning |
| --- | --- | --- |
| red | `back-edge` | The layering is crossed. Either a later step invalidates or constrains an earlier one (arrow points back), or an optional optimization feeds a required placement step (arrow points forward) |
| amber | `own-judgment` | Answers "which chunk imports which" its own way instead of reading one shared definition, or runs a judgment narrower than what it changes |
| amber | `direct-write` | Assigns fields on shared state directly; no value stands for "one change" |
| green | `bounded-loop` | A loop, but a local one: monotone, bounded, nothing half-done escapes |

`(strict)` in a title marks a step that exists only under
`output.strictExecutionOrder`, which is off by default.

A circled number marks a concept that exists in both rows but sits somewhere
materially different — find the same number in the other row to see where it
moved. ① interop code position, ② `avoidRedundantChunkLoads`, ③ where the
runtime ends up, ④ `chunkModulesOrder`, ⑤ `ChunkLoadGraph`.

## What the two rows are meant to show

**Top row.** Four amber boxes are four separate answers to the same question —
the atom graph `avoidRedundantChunkLoads` builds, the temporary graph
`mergeCommonChunks` builds, the reachability walk runtime merging does, and the
real post-lowering edges. The shared `ChunkLoadGraph` is the last node on the
right, computed after sealing, which is to say after every judgment above has
already run against its own answer. The cylinder collects the direct writes:
every writer assigns fields on it.

**Bottom row.** No crossing of the layering. `ChunkLoadGraph` is computed right
after placement and read by every judgment. Every change to the placement is a
value proposed to one judge, and the only arrow that runs backwards is that
judge applying an accepted change.

The same concept keeps the same name in both rows, so the interesting part is
where a name sits. `Compute load sets` and `Automatic code splitting`, for
instance: today three things run between them, and one of them is optional and
rewrites the load sets; in the intended model only manual code splitting sits
between them, and it has to, because a manual group that fails its size rules
hands its modules back.

## Files

```
src/spec.c4        notation: element kinds, markers, relationship kinds
src/1-today.c4     what main does
src/2-proposal.c4  the intended layering
src/views.c4       the two views
```

In `index` the `include` order is reversed (`proposed.*, now.*`) because the
row listed last renders on top.

## Commands

```bash
npm run validate   # syntax and references
npm run fmt        # format
npm run png        # export out/*.png
```

`npm run png` renders through a headless chromium; the first run needs
`./node_modules/.bin/playwright install chromium-headless-shell`. Its canvas
clips at 16384px. Both views use `size sm` and a tightened `autoLayout`, which
keeps `index` around 14.8k wide; past the limit the right edge of the export is
cut. `npm run dev` has no such limit.

## Provenance

The models were written by reading rolldown's source at `6a54a0531`. The
descriptions are an outside reading of that code, not a maintainer's statement
of intent.

One premise the intended model rests on is not verified: that the wrapping
analysis only ever changes the wrap set and never module ownership. What was
read supports it — the trigger file adds a chunk holding no module and changes
entry routing — but `order_wrapping.rs` was not read exhaustively. If that
premise fails, wrapping feeds back into placement and the loop in the bottom
row stops being a local one.
