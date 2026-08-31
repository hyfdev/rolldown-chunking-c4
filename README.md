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

**Bottom row.** No crossing of the layering. The splitting rules write
`BaseChunkPlacement` once and never touch it again — the finest partition anyone
gets, since every change after it merges or reroutes and none of them splits.
From there the placement moves only as `ChunkPlacement`, and its only two
writers are the same judgment applying a change it accepted. `ChunkLoadGraph` is
computed from the placement and read by every judgment. Today's row has no
counterpart to the base at all: there is one mutable structure and eleven
writers assign fields on it, so no value stands for "the placement the rules
produced".

The same concept keeps the same name in both rows, so the interesting part is
where a name sits. `Compute load sets` and `Automatic code splitting`, for
instance: today three things run between them, and one of them is optional and
rewrites the load sets; in the intended model only manual code splitting sits
between them, and it has to, because a manual group that fails its size rules
hands its modules back.

## The optimizations, and that these are all of them

Four passes change the placement for optimization. Everything else that writes
it is a required rule — entry chunk creation, `preserveModules`, manual and
automatic code splitting — or the strict trigger facade.

| Pass in `main` | Where it sits |
| --- | --- |
| `optimize_dynamic_entry_bits` | ② `avoidRedundantChunkLoads` |
| `try_insert_common_module_to_exist_chunk` | `mergeCommonChunks: shared modules` |
| `optimize_facade_entry_chunks` | `mergeCommonChunks: empty facades` |
| `try_merge_runtime_chunk`, `sweep_unused_runtime_module` | ③ |

The intended row carries one more, `inlineCommonChunks`, which `main` does not
have: a common chunk that does not earn a file dissolves into its consumers,
with every module still evaluating exactly once. It sits between
`BaseChunkPlacement` and `ChunkPlacement` — not as a change to the placement but
as how the working placement is derived from the base. It is optional, and a
base that changes with configuration is not a fixed reference for anything,
which is the same defect ② marks in today's row.

The list of passes in `main` closes on a check anyone can repeat: grep the generate stage for the
writes that move a module between chunks, add a chunk, or mark one removed
(`add_module_to_chunk`, `module_to_chunk[..] =`, `add_chunk`,
`post_chunk_optimization_operations.insert`). Six files answer, and each write
belongs to one of the passes above or to a required rule.

③ is the reason the numbering earns its keep. In the intended model it is one
node: merge the runtime into its consumer, drop it if nothing references it, or
leave it standalone — one question, asked once, last, because wrapping is what
completes the demand. Today it is two passes with the wrapping analysis between
them, and the second one invalidates an execution order the pipeline already
assigned.

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
clips at 16384px. Both views use `size sm`, and `index` a tightened
`autoLayout`. It renders 16.2k wide against that 16.4k ceiling, so it is one
node away from a cut right edge. Rank separation is spent: dropping it from
`40 30` to `14 14` bought 210px, because the width is node width times rank
count, not the gaps. The levers left are fewer ranks or a narrower node size.
`npm run dev` has no such limit.

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
