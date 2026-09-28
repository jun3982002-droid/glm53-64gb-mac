# DS4 patch: approximate expert routing with an exact warm-up

This directory contains the code change described in [the 2026-09-28 results](../results/2026-09-28-model-comparison-and-derailment-fix.md): making the first N decode steps of every answer exact in the approximate GLM-5.3 modes. The change is published as a patch against the upstream DS4 runtime.

| File | What it is |
|---|---|
| [`ds4-glm53-approx-routing-warmup.patch`](./ds4-glm53-approx-routing-warmup.patch) | The patch. |
| [`LICENSE`](./LICENSE) | The upstream DS4 license file (MIT), unchanged. |

## Base revision

- Upstream: [antirez/ds4](https://github.com/antirez/ds4)
- Commit: `0aaea5a238fb41a35106a551e73c8409dfb751ac`

## What the patch contains

The warm-up only turns the approximation off for a while, so it has no effect without the approximation itself. The patch therefore contains both.

1. **Approximate expert routing** for the single-token decode path with SSD streaming:
   - cache-aware drop (`DS4_GLM_CACHE_AWARE_MIN`, `_TAU`, `_MASS`)
   - router-probability substitution (`DS4_GLM_SUBST=router`)
2. **The warm-up fix** (`DS4_GLM_APPROX_WARMUP_TOKENS`). It is wired into two decode loops:
   - the CLI's greedy generation path (`--temp 0`, without MTP drafting, tensor parallelism or a distributed run);
   - the server's per-request decode loop.

   Other paths are **not** wired: the CLI with a temperature above 0 (the CLI default is 1.0), interactive chat in the CLI, and perplexity scoring. If the variable is set there, the step counter never advances, so every step runs exact and the approximation is effectively off.

Everything is off by default. With none of the variables set, the new code paths are skipped. In our check, exact output with no variables set was byte-identical to unpatched upstream (see [Checks performed](#checks-performed)).

These parts of our development runtime are **not** included:
- two-drive striped reads
- next-layer expert prefetch
- the launcher and server scripts
- tracing and measurement code
- other experiments

The speeds in the results notes were measured with striped reads, and some modes also used prefetch. This patch alone does not reproduce those speeds. The `jevq6s` mode used prefetch, so the patch can only run it without prefetch.

## Environment variables

These apply only when running with `--ssd-streaming` on Metal.

| Variable | Meaning |
|---|---|
| `DS4_GLM_CACHE_AWARE_MIN=m` | Expert slots ranked below `m` always load. A lower-ranked slot is used only if its expert is already cached; otherwise it is dropped (no SSD read) and given weight 0. |
| `DS4_GLM_CACHE_AWARE_TAU=tau` | With `MIN`, a missing slot whose normalized router weight is at least `tau` is loaded anyway. |
| `DS4_GLM_CACHE_AWARE_MASS=p` | Mass-coverage drop. Cached slots are always used. Missing slots are loaded in rank order until the used router-weight mass reaches `p`, and the rest are dropped. The top-ranked slot is always used. Replaces `MIN`/`TAU` when set. |
| `DS4_GLM_SUBST=router` | A dropped slot is replaced by the already-cached expert with the highest router probability that this token does not already use. The dropped slot's weight is kept. If none is cached, the slot keeps weight 0. |
| `DS4_GLM_APPROX_WARMUP_TOKENS=N` | In the wired paths, the four settings above are turned off for the first `N` decode steps of each generation, then resume. The first generated token comes from prompt processing, which is already exact. So with `N=16`, generated tokens 2–17 are exact. |

The modes named in the results notes correspond to:
- `fast4s`: `DS4_GLM_CACHE_AWARE_MIN=2 DS4_GLM_CACHE_AWARE_TAU=0.15 DS4_GLM_SUBST=router`
- `jevq6s` without prefetch: `DS4_GLM_CACHE_AWARE_MASS=0.6 DS4_GLM_SUBST=router`

In our tests we used `DS4_GLM_APPROX_WARMUP_TOKENS=16` with these modes.

## Build

```bash
git clone https://github.com/antirez/ds4
cd ds4
git checkout 0aaea5a238fb41a35106a551e73c8409dfb751ac
git apply /path/to/ds4-glm53-approx-routing-warmup.patch
make ds4 ds4-server
```

Example run (the model file is not distributed here):

```bash
DS4_GLM_CACHE_AWARE_MIN=2 DS4_GLM_CACHE_AWARE_TAU=0.15 DS4_GLM_SUBST=router \
DS4_GLM_APPROX_WARMUP_TOKENS=16 \
./ds4 --model /path/to/GLM-5.3-Full-IQ2_XXS.gguf --metal --ssd-streaming \
  --ctx 8192 --power 100 --nothink --temp 0 --seed 42 --tokens 64 --prompt-file prompt.txt
```

With the warm-up active, the CLI prints a line like `GLM approx-warmup: forced exact for 16/63 decode step(s)` at the end.

## Checks performed

These checks ran on 2026-09-29 on the test machine (Apple M4 Max, 64 GiB, macOS, Metal).

- **Build:**
  - We cloned the base commit fresh from GitHub and applied this patch file with `git apply` (exit code 0).
  - `make ds4 ds4-server` exited with code 0. The build log had no warning or error lines.
- **CLI output comparison:**
  - Conditions: GLM-5.3 Full IQ2_XXS, the same 95-token Japanese prompt, 64 generated tokens, `--ctx 8192 --power 100 --nothink --temp 0 --seed 42`. No stripe manifest. In the patched and upstream runs, no `DS4_` environment variables were set other than those in the table.
  - The patched build's output was compared by SHA-256 with two references:
    - unpatched upstream (exact only);
    - our full development build, with its extra features (striped reads, prefetch, tracing) off.

| Setting | vs. unpatched upstream | vs. development build |
|---|---|---|
| exact (no variables) | identical | identical |
| `fast4s` | — | identical |
| `fast4s` + `DS4_GLM_APPROX_WARMUP_TOKENS=16` | — | identical |
| `MASS=0.6` + `SUBST=router` | — | identical |
| `MASS=0.6` + `SUBST=router` + `DS4_GLM_APPROX_WARMUP_TOKENS=16` | — | identical |

- **Warm-up active:** both warm-up runs reported `forced exact for 16/63 decode step(s)`. Their outputs differed from the same settings without the warm-up.
- **Server path:** we started the patched `ds4-server` with the `fast4s` settings plus `DS4_GLM_APPROX_WARMUP_TOKENS=16` (default mode, not `--batched-sessions`) and sent two short chat requests.
  - Both answers were correct (3776 and 東京).
  - The server logged `forced exact for 3 decode step(s)` and `forced exact for 1 decode step(s)`: both answers were shorter than the 16-step window.
  - The count for the second request started again from zero, so the per-request reset works.
- **Not checked with this build:**
  - perplexity scoring, the CLI with a temperature above 0, and CLI interactive chat (none of these are wired)
  - `ds4-server --batched-sessions`
  - answers longer than 64 tokens
  - other prompts
  - other hardware
  - non-Metal backends

## Limitations

- **Metal (macOS) only.** Only the Metal build was compiled and run.
  - The patch changes a function signature that the upstream CUDA and ROCm backends also define or declare (`ds4_cuda.cu`, `rocm/ds4_rocm_current_api_compat.cuh`, `rocm/ds4_rocm_glm.cuh`), and does not update those files.
  - The new warm-up calls in `ds4.c` and `ds4_server.c` refer to functions defined only in the Metal backend (`ds4_metal.m`).
  - Non-Metal builds are therefore expected to fail.
- **Single-generation counter.** The warm-up counter is process-wide, so it assumes one generation decodes at a time.
  - This holds for the CLI, and for `ds4-server` in its default mode, where decoding is serialized.
  - With `--batched-sessions`, several sessions decode concurrently and share the counter, so the warm-up window is not tracked correctly. That mode was not tested.
- **Experimental.** The approximate modes change the model's computation, and their output can depend on earlier requests (cache state).
  - In our earlier tests (see the results notes), the warm-up fixed one reproduced derailment. That scenario was not rerun with this patch build.
  - The warm-up does not protect later parts of an answer.

## License

This patch is released under the MIT License, the same license as upstream DS4. [`LICENSE`](./LICENSE) is the upstream DS4 license file, unchanged, including its copyright notices.
