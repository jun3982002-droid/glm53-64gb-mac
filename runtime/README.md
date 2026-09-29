# DS4 patches: single-drive warm-up and full fast5 two-drive setup

This directory contains two alternative patches against the same upstream DS4 revision. The single-drive patch adds approximate GLM-5.3 expert routing with an exact warm-up. The consolidated full fast5 patch also includes two-drive striped reads, per-drive readahead, and the builder/verifier for its sidecar manifest. The model weights are not included in either patch.

| File | What it is |
|---|---|
| [`ds4-glm53-approx-routing-warmup.patch`](./ds4-glm53-approx-routing-warmup.patch) | Single-drive routing and exact warm-up patch. |
| [`ds4-glm53-full-fast5-two-drive.patch`](./ds4-glm53-full-fast5-two-drive.patch) | Consolidated fast5 patch with optional two-drive striping and stripe-aware readahead. |
| [`LICENSE`](./LICENSE) | The upstream DS4 license file (MIT), unchanged. |

Use **one patch only** on a clean checkout of the pinned upstream revision. The single-drive patch is smaller. Use the consolidated patch to reproduce the two-drive recipe below.

## One-drive quick start

This recipe builds the single-drive patch on the exact upstream revision it targets, downloads the GLM-5.3 Full IQ2_XXS GGUF through DS4's documented model target, and runs a short generation. It is for an **Apple Silicon Mac using Metal and SSD streaming**. The patch was built and checked on an M4 Max with 64 GiB of unified memory; other Macs and backends have not been checked here.

The model file is about **197 GiB**. The DS4 downloader puts it under `gguf/` in the cloned repository, so clone DS4 onto the SSD you want to hold the model and allow extra free space for download overhead. Model weights are not stored in this GitHub repository.

```bash
git clone https://github.com/antirez/ds4.git
cd ds4
git checkout 0aaea5a238fb41a35106a551e73c8409dfb751ac
curl -L --fail -o ds4-glm53-approx-routing-warmup.patch \
  https://raw.githubusercontent.com/jun3982002-droid/glm53-64gb-mac/efee16f98370900d42c5e9a9a0214fbf44274390/runtime/ds4-glm53-approx-routing-warmup.patch
git apply ds4-glm53-approx-routing-warmup.patch
make ds4
./download_model.sh glm53-full-q2
```

Run a short answer with the `fast5` cache-aware thresholds and the 16-token exact warm-up. The public patch accepts these settings as environment variables; it does not provide a launcher option named `fast5`.

```bash
DS4_GLM_CACHE_AWARE_MIN=1 \
DS4_GLM_CACHE_AWARE_TAU=0.18 \
DS4_GLM_APPROX_WARMUP_TOKENS=16 \
./ds4 --model gguf/GLM-5.3-UD-IQ2_XXS_RoutedIQ2XXS_blk78Q2K.gguf \
  --metal --ssd-streaming --ctx 8192 --power 100 --nothink \
  --temp 0 --seed 42 --tokens 64 \
  -p "Explain why the sky appears blue in two sentences."
```

This setting drops some cache-missing experts without substituting a different expert. The first 16 decode steps are exact, then approximation resumes. The single-drive patch has no striping. With the included Japanese benchmark prompt, one run on the M4 Max measured 2.56 tok/s generation and 4.01 tok/s prefill. A matched run using the consolidated patch with one drive measured 2.59 tok/s with warm-up 16; the full one-/two-drive comparison is below. Each result is a short, single-run speed measurement, not a performance guarantee. To use the exact reference path, run the same command without the three `DS4_GLM_...` environment variables.

## Full fast5 quick start: two-drive striping

This is the complete recipe for the higher-speed two-drive configuration. It builds the consolidated patch from a clean pinned upstream checkout, downloads the model onto the first drive, makes a sidecar containing 59% of each routed-expert slice on a second drive, verifies that sidecar, and runs the `fast5` cache-aware settings. It was measured on an Apple M4 Max with 64 GiB, macOS, Metal, one external NVMe model drive, and the internal SSD as the second drive. Other hardware, SSDs, and backends have not been checked.

The model is **196.58 GiB**. Allow at least that much free space on the first drive plus download overhead. The generated sidecar is **111,333,605,376 bytes (103.68 GiB)**, so the second drive needs at least that much free space, plus room for the manifest and other files. The builder reads the model metadata and routed-expert data ranges; it does not modify the original GGUF. Neither the GGUF, sidecar, nor manifest is hosted in this repository.

Clone DS4 onto the drive that will hold the model. Apply **only** the consolidated patch (do not first apply the single-drive patch):

```bash
git clone https://github.com/antirez/ds4.git
cd ds4
git checkout 0aaea5a238fb41a35106a551e73c8409dfb751ac
curl -L --fail -o ds4-glm53-full-fast5-two-drive.patch \
  https://raw.githubusercontent.com/jun3982002-droid/glm53-64gb-mac/main/runtime/ds4-glm53-full-fast5-two-drive.patch
git apply ds4-glm53-full-fast5-two-drive.patch
make ds4
./download_model.sh glm53-full-q2
curl -L --fail -o fast5-benchmark-prompt-ja.txt \
  https://raw.githubusercontent.com/jun3982002-droid/glm53-64gb-mac/main/results/fast5-benchmark-prompt-ja.txt
```

Set the model and second-drive paths. Replace the second-drive example with a directory on a physically separate storage device. Keep the model directory on the first drive:

```bash
MODEL="$PWD/gguf/GLM-5.3-UD-IQ2_XXS_RoutedIQ2XXS_blk78Q2K.gguf"
SECOND_DRIVE_DIR="/path/on/second-drive"
mkdir -p "$SECOND_DRIVE_DIR"
SIDECAR="$SECOND_DRIVE_DIR/glm53.stripe.part0.bin"
MANIFEST="$SECOND_DRIVE_DIR/glm53.stripe.manifest.json"
PROMPT="$PWD/fast5-benchmark-prompt-ja.txt"
```

Build and verify the stripe files. `--part PATH:0.59` writes 59% of every routed-expert slice into the sidecar; `--tail-fraction 0.41` leaves the other 41% in the original GGUF. The split is aligned to 16,384-byte boundaries. The verified GLM file had 58,368 routed-expert slices, totaling 176.98 GiB; the sidecar contains 103.68 GiB. The manifest records the byte ranges and files for DS4:

```bash
python3 tools/ds4_stripe_build.py build \
  --gguf "$MODEL" \
  --part "$SIDECAR:0.59" \
  --tail-fraction 0.41 \
  --align 16384 \
  --manifest "$MANIFEST"
python3 tools/ds4_stripe_build.py verify --manifest "$MANIFEST"
```

The 0.59/0.41 split was selected after an 8-thread concurrent random-read check on this machine measured about 5.32 GB/s on the internal SSD and 3.67 GB/s on the external SSD. Re-measure the two devices before choosing a ratio on different hardware; do not assume that 0.59 is optimal elsewhere.

The matched comparison below used the **same consolidated binary** for both storage layouts. For the one-drive baseline, leave the stripe manifest unset. For the two-drive run, set it to the verified manifest and keep four reader threads per part. In both commands, the 16-token exact warm-up is the recommended setting. It fixed one previously reproduced start-of-answer derailment sequence in an earlier 10/10 check, but that does not establish general protection; it also reduces speed. The benchmark prompt is included in this repository. Other conditions were GLM-5.3 Full IQ2_XXS, M4 Max 64 GiB, Metal, `--ctx 8192 --power 100 --nothink --temp 0 --seed 42 --tokens 64`, `MIN=1`, `TAU=0.18`, no MASS, no substitution, no next-layer expert prefetch, and no concurrent `ds4-server`.

One-drive baseline:

```bash
env -u DS4_STRIPE_MANIFEST \
  -u DS4_STRIPE_THREADS_PER_PART \
  -u DS4_GLM_CACHE_AWARE_MASS \
  -u DS4_GLM_SUBST \
  -u DS4_XLAYER_PREFETCH \
  DS4_GLM_CACHE_AWARE_MIN=1 \
  DS4_GLM_CACHE_AWARE_TAU=0.18 \
  DS4_GLM_APPROX_WARMUP_TOKENS=16 \
  ./ds4 --model "$MODEL" --metal --ssd-streaming \
    --ctx 8192 --power 100 --nothink --temp 0 --seed 42 --tokens 64 \
    --prompt-file "$PROMPT"
```

Full two-drive fast5 run:

```bash
DS4_STRIPE_MANIFEST="$MANIFEST" \
DS4_STRIPE_THREADS_PER_PART=4 \
DS4_GLM_CACHE_AWARE_MIN=1 \
DS4_GLM_CACHE_AWARE_TAU=0.18 \
DS4_GLM_APPROX_WARMUP_TOKENS=16 \
env -u DS4_GLM_CACHE_AWARE_MASS \
  -u DS4_GLM_SUBST \
  -u DS4_XLAYER_PREFETCH \
  ./ds4 --model "$MODEL" --metal --ssd-streaming \
    --ctx 8192 --power 100 --nothink --temp 0 --seed 42 --tokens 64 \
    --prompt-file "$PROMPT"
```

For a speed-only comparison without the exact warm-up, set `DS4_GLM_APPROX_WARMUP_TOKENS=0` in each command. That produced the higher 5.58 and 6.78 tok/s observations below, but disabling the warm-up can expose the known rare derailment. Do not use those peak figures as a quality claim; output quality was not re-evaluated in the matched speed run.

### Matched one-drive and two-drive results

All four measurements below used the same consolidated build, model, prompt, and generation settings. Each cell is one run on 2026-09-29. “No warm-up” means `DS4_GLM_APPROX_WARMUP_TOKENS=0`; “warm-up 16” means the first 16 decode steps run exact (the run logged 16/63 exact steps).

| Storage | Warm-up | Prefill | Generation |
|---|---:|---:|---:|
| One drive | none | 3.99 tok/s | 5.58 tok/s |
| One drive | 16 steps | 3.99 tok/s | 2.59 tok/s |
| Two-drive striping (`r=0.59`, 4 threads per part) | none | 8.22 tok/s | 6.78 tok/s |
| Two-drive striping (`r=0.59`, 4 threads per part) | 16 steps | 8.28 tok/s | 4.19 tok/s |

These are single-run measurements on one machine and one prompt. The no-warm-up generation results were faster but have a known rare output-derailment risk; the speed-only run did not score output quality. The separate NLL result for fast5 was +21.48% on one 991-token text (959 tokens scored), not a general quality score. No hash of the model file was recorded. The full two-drive patch reproduced the mechanism and this benchmark on this machine; it does not guarantee the same speed on other hardware. SSD speed, cache state, filesystem behavior, model placement, and GPU/runtime work can change end-to-end results.

## Base revision

- Upstream: [antirez/ds4](https://github.com/antirez/ds4)
- Commit: `0aaea5a238fb41a35106a551e73c8409dfb751ac`

## What the single-drive patch contains

The warm-up only turns the approximation off for a while, so it has no effect without the approximation itself. The patch therefore contains both.

1. **Approximate expert routing** for the single-token decode path with SSD streaming:
   - cache-aware drop (`DS4_GLM_CACHE_AWARE_MIN`, `_TAU`, `_MASS`)
   - router-probability substitution (`DS4_GLM_SUBST=router`)
2. **The warm-up fix** (`DS4_GLM_APPROX_WARMUP_TOKENS`). It is wired into two decode loops:
   - the CLI's greedy generation path (`--temp 0`, without MTP drafting, tensor parallelism or a distributed run);
   - the server's per-request decode loop.

   Other paths are **not** wired: the CLI with a temperature above 0 (the CLI default is 1.0), interactive chat in the CLI, and perplexity scoring. If the variable is set there, the step counter never advances, so every step runs exact and the approximation is effectively off.

Everything is off by default. With none of the variables set, the new code paths are skipped. In our check, exact output with no variables set was byte-identical to unpatched upstream (see [Checks performed](#checks-performed)).

These parts of the full development runtime are **not** included in the single-drive patch:
- two-drive striped reads
- next-layer expert prefetch
- the launcher and server scripts
- tracing and measurement code
- other experiments

The consolidated full fast5 patch listed at the top includes two-drive striping and stripe-aware readahead, but still does not include next-layer expert prefetch. Neither patch includes the experiment launcher or server scripts. The `jevq6s` mode used next-layer prefetch in some experiments, so these patches can only run it without that feature.

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
- `fast5` cache-aware settings: `DS4_GLM_CACHE_AWARE_MIN=1 DS4_GLM_CACHE_AWARE_TAU=0.18`, with MASS and SUBST unset

The full fast5 procedure and its matched one-/two-drive results are listed above. The single-drive patch supports the cache-aware `fast5` settings but does not include striping. The consolidated patch includes the striping implementation and uses the same settings for the comparison above.

## Build

The [one-drive quick start](#one-drive-quick-start) and [full fast5 two-drive quick start](#full-fast5-quick-start-two-drive-striping) each contain pinned clone, patch, build, model download, and run commands. Apply only one patch to a clean checkout. Both build the `ds4` CLI; add `make ds4-server` if you also want the server binary.

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
- **Earlier `fast5` check on the single-drive patch (different, unpublished prompt):** we built a clean checkout of base commit `0aaea5a` plus the single-drive patch with `make ds4` (exit code 0). On an M4 Max with 64 GiB, Metal, one external NVMe model drive, no stripe manifest, no next-layer prefetch, and no `ds4-server` running concurrently, we used `MIN=1`, `TAU=0.18`, no MASS, no SUBST, and `DS4_GLM_APPROX_WARMUP_TOKENS=16`. With a fixed 95-token Japanese prompt, `--ctx 8192 --power 100 --nothink --temp 0 --seed 42 --tokens 64`, three runs measured prefill at 2.19 / 2.20 / 2.20 t/s and generation at 2.57 / 2.59 / 2.60 t/s (median 2.59). Each run reported `forced exact for 16/63 decode step(s)`. That prompt is not the public prompt included above, so these runs are not directly comparable to the matched one-/two-drive table. No model hash was recorded. The model weights were read from the existing file and were not modified.
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

- **Metal (macOS) only.** Only the Metal build was compiled and run; treat both patches as Metal-only.
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

Both patches are released under the MIT License, the same license as upstream DS4. [`LICENSE`](./LICENSE) is the upstream DS4 license file, unchanged, including its copyright notices.
