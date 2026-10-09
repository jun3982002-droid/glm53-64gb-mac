# Large MoE Inference on a 64 GB Mac

**Can a 64 GB Mac run a model whose weights are much larger than memory—and what trade-offs make it practical?**

This project measures SSD-streamed inference for **GLM-5.3 Full, IQ2_XXS quantization** (196.58 GiB model file) on an Apple M4 Max with 64 GiB of unified memory. The repository includes both a one-drive patch and a consolidated two-drive fast5 patch for the [DS4](https://github.com/antirez/ds4) runtime. The two-drive patch splits only routed-expert byte ranges between an internal SSD sidecar and the external model file; it does not modify or redistribute the model. The same idea is also reported below for DeepSeek V4.1 Flash on a separate experimental runtime.

## Run it on an Apple Silicon Mac

This repository includes two copyable runtime setups in [`runtime/README.md`](runtime/README.md): single-drive approximate routing, and the full fast5 two-drive version with a sidecar builder and verifier. The full GLM-5.3 Q2 model is about 197 GiB, so it is downloaded directly with DS4's model downloader and is **not** hosted here. The two-drive method and benchmark settings are documented below.

Start with the [full fast5 two-drive quick start](runtime/README.md#full-fast5-quick-start-two-drive-striping) to reproduce the combined patch and striped setup, or use the [one-drive quick start](runtime/README.md#one-drive-quick-start) as the storage baseline. The full recipe pins the exact upstream DS4 revision, builds the runtime, downloads the model, creates and verifies the sidecar, and runs the included benchmark prompt.

## Recent DeepSeek V4.1 Flash decode results (2026-10-09)

- **93,030-token DS4 benchmark:** across completed runs with a 34 GB cache, 512-token decode averaged 15.11–15.26 tok/s; the fastest run reached 15.37 tok/s in its steady-state portion.
- **Repeated-output trial:** a separate API request with 22,034 prompt tokens and 2,048 generated tokens reached a peak of **17.39 tok/s in a rolling 50-token window**. The full decode measured 15.93 tok/s in the server log and 15.35 tok/s at the client. The request ended at its token limit while generating in the reasoning channel, without the requested visible output; this is speed evidence, not an answer-quality result.

These are different prompts and measurement paths. The 17.39 tok/s figure is the highest short-window speed observed in the repeated-output trial, not the 93k benchmark average.

## Selected results

These are dated measurements from specific experiments, not a general performance guarantee. The public Japanese benchmark prompt is included at [`results/fast5-benchmark-prompt-ja.txt`](results/fast5-benchmark-prompt-ja.txt).

| Experiment | Measured result | Conditions and limits |
|---|---|---|
| One-drive routing patch, `fast5` settings + warm-up 16, 2026-09-29 | Generation 2.56 tok/s; prefill 4.01 tok/s | M4 Max, 64 GiB, Metal, one external NVMe model drive, included Japanese benchmark prompt, 64-token limit, one run. `MIN=1`, `TAU=0.18`, MASS/substitution/next-layer prefetch unset. |
| Consolidated full fast5 patch, matched one-/two-drive and warm-up check, 2026-09-29 | One drive: 5.58 tok/s without warm-up, 2.59 with warm-up 16. Two-drive striping: 6.78 without warm-up, 4.19 with warm-up 16. | Same M4 Max, model, public prompt, binary, `MIN=1`, `TAU=0.18`, 64-token limit; one run per cell. Two-drive uses `r=0.59`, `DS4_STRIPE_THREADS_PER_PART=4`, and stripe-aware readahead. Prefill was 3.99 / 3.99 tok/s (one drive) and 8.22 / 8.28 tok/s (two drives), no-warm-up/warm-up 16. Speed-only runs; output quality was not re-evaluated. Turning warm-up off can expose a known rare derailment. |
| DeepSeek V4.1 Flash, expert data split 59:41 across the internal and external SSD, 2026-09-26 | Unsplit 4.573 / 4.558 / 4.467 tok/s (mean 4.53) → split 5.761 / 5.089 / 5.067 (mean 5.31, +17%). Output was bit-identical (SHA-256). | Experimental SSD-streaming runtime (not included here). Same M4 Max, 35-token prompt + 200 generated tokens, three runs per setting. Later checks (2026-09-30) with the same 59:41 layout gave 5.273 / 5.261 / 5.286 tok/s on a different fixed prompt and 3.54–3.61 on another. Speed depends on the prompt, and no matched one-drive comparison has been repeated on the current layout, so whether the +17% still holds there is not settled. Details: [two SSDs, split by drive speed](#reading-experts-from-two-ssds-split-by-drive-speed). |
| `jevq6s` decode, 2026-09-28 | 3.62 / 3.59 tok/s across two runs; exact measured about 2.1 tok/s | Experimental approximate mode combining cache-aware MASS, resident-expert substitution, and next-layer prefetch. In separate NLL evaluations, +2.06% vs exact on one Japanese text (959 tokens scored) and +2.03% on another (2,222 tokens scored). These two texts do not establish general response quality. |
| Exact vs. `fast4s` decode, 2026-09-27 | 1.955 vs. 5.555 generated tokens/s (2.84× ratio of the two-run means) | Same fixed Japanese prompt, 64 generated tokens, 2 runs per mode. `fast4s` is approximate; it substitutes already-resident experts for selected cache-missing experts. This is not an apples-to-apples quality comparison. |
| `fast4p` prefill, 2026-09-27 | 4.535 → 5.785 input tokens/s (+27.56%); decode stayed at 5.59 → 5.615 output tokens/s | 95-token input, two paired runs. `fast4p` changes prefill behavior while retaining the `fast4s` decode configuration. |
| Exact-mode memory threshold, 2026-09-28 | 2.135 tokens/s with static weights locked vs. 1.85 tokens/s when they were pageable | Two runs per boundary setting, plus one control run. The normal automatic setting keeps the weights locked; increasing the expert cache past the boundary made things slower. |
| Three-model comparison, 2026-09-28 | 24 easier questions: GLM `jevq6s` 24/24, Qwen3.8 Flash Next 22/24, DeepSeek V4.1 Flash 19/24 (18/24 if one truncated answer is counted as wrong). 20 harder questions: Qwen 20/20, GLM `jevq6s` 18/20, DeepSeek 13/20. | Japanese questions, graded automatically, each run once per model; temperature 0, thinking off. The sets are small, so a one- or two-question gap is a tie. Qwen answered roughly 20–50× faster per question. All three were quantized builds, and GLM and DeepSeek ran on experimental SSD-streaming runtimes; these are results as run here, not for the full-precision models. Details: [2026-09-28 results](results/2026-09-28-model-comparison-and-derailment-fix.md) |
| Derailment in `jevq6s` and a fix, 2026-09-28 | After one specific sequence of 9 earlier questions, `jevq6s` answered a logic puzzle with an unrelated article, and this reproduced exactly. Running the first 16 decode steps of each answer without approximation fixed it (10/10 on the reproducing sequence). | Only one reproduced case. It does not protect later parts of an answer. Cost on 64-token answers: −15% decode speed for `jevq6s` and −30% for `fast4s` (one run each). For `jevq6s` at 256 tokens: −4.4% (two-run mean on the test build). Details: [2026-09-28 results](results/2026-09-28-model-comparison-and-derailment-fix.md) |

### Quality notes

- In a separate `fast4s` evaluation, NLL was **5.98% higher than exact** on one 991-token text (959 tokens scored). This is a change in one language-modeling metric, not “6% quality loss,” accuracy, or a broad measure of usefulness.
- In a separate `fast4p` comparison with `fast4s`, NLL was +0.8128% and +1.0984% on two texts (991 tokens each; 150-token prefix, 841 continuation tokens scored). A few short JSON, arithmetic, and Japanese prompts were also checked. Earlier longer checks included an unfinished Japanese response and an arithmetic answer that was wrong in both modes; later short-answer checks completed, but these small samples do not establish general task quality.
- Exact-mode output hashes matched the reference in the tested cases. That claim applies to those checks only.
- Approximate modes choose experts partly from what is currently in the memory cache, so the same prompt can give different output depending on earlier requests. NLL averages cannot show rare failures of this kind. One such derailment was reproduced and fixed; see the [2026-09-28 results](results/2026-09-28-model-comparison-and-derailment-fix.md). In those runs, the exact mode did not derail.

## Choose a mode

The experimental launcher accepted these modes by name on the test machine. The repository includes a single-drive routing patch and a consolidated two-drive fast5 patch; the full development launcher is not included. The `fast5` settings and reproducible two-drive setup are in [`runtime/README.md`](runtime/README.md).

| Mode | Decode speed | NLL difference vs. exact | When to choose |
|---|---:|---:|---|
| `exact` | ~2.1 tok/s | 0 (reference) | Use the reference path when fidelity is the priority. Output hashes matched the reference in tested cases only. |
| `jevq6s` | ~3.6 tok/s | +2.03% / +2.06% on two Japanese texts | Quality-priority starting point; the current recommended preset for these tests. |
| `jevq5s` | ~3.8 tok/s | +3.69% / +3.81% on the same two texts | A little more speed, with a larger measured NLL difference in those texts. |
| `fast4q` | ~4.5 tok/s | +4.5% in the reported evaluation | Middle-speed option. |
| `fast4s` | 5.3–5.5 tok/s | +5.98% on one 991-token text (959 scored) | Choose when decode speed matters more. |
| `fast5` | 6.78 tok/s in the two-drive, no-warm-up run; 4.19 with warm-up 16 | +21.48% on one 991-token text (959 scored) | `MIN=1`, `TAU=0.18`, no MASS, substitution, or next-layer prefetch; two-drive patch with `r=0.59`. NLL is from a separate evaluation. The no-warm-up speed is experimental and can expose a known rare derailment. |

The matched check above uses the same consolidated patch with the manifest unset for one drive and set for two-drive striping. With the included prompt, striping measured 6.78 tok/s without warm-up and 4.19 with warm-up 16, versus 5.58 and 2.59 on one drive. Each cell was measured once. The separate one-drive patch does not include striping; apply the consolidated patch to reproduce the two-drive procedure.

These are project snapshots from different runs and evaluation conditions, not a single apples-to-apples benchmark. NLL is a language-modeling metric, not a percentage score for answer quality or accuracy; the text evaluations are small.

The speeds in this table were measured before a start-of-answer safeguard was added. The runtime patch can run the first 16 decode steps exactly, then resume approximation. In the matched `fast5` check above, warm-up reduced speed from 5.58 to 2.59 tok/s on one drive and from 6.78 to 4.19 tok/s with two-drive striping (one run per cell). It is a safety/performance tradeoff; disabling it can expose a known rare derailment. On other 64-token runs, the same warm-up cost about 15% for `jevq6s` and 30% for `fast4s` (one run each: 3.85 → 3.28 and 5.54 → 3.88 tok/s). The 3.85 differs from the ~3.6 in the table: with the same settings, `jevq6s` measured 3.59–3.62 tok/s in an earlier session and 3.85–3.86 tok/s in a later one, and the cause of that gap was not found.

## How the approximation works

The `jevq6s` mode combines cache-aware expert selection, resident-expert substitution, and next-layer prefetch. The `fast4s` mode can skip selected experts that are not in the memory cache, then use a suitable expert that is already resident. These modes reduce some SSD reads but change the computation. `exact` remains available as the reference mode. Approximate output is not described as lossless. Because the choice depends on the cache contents, approximate output can depend on earlier requests, not only on the prompt. In the launcher used for these tests (not yet published), the approximate modes now run the first 16 decode steps of each answer without approximation, to protect the start of the answer; `fast4p` runs on a separate binary and is not covered (see the [2026-09-28 results](results/2026-09-28-model-comparison-and-derailment-fix.md)).

## Reading experts from two SSDs, split by drive speed

In these experiments the expert weights of a large MoE model stay on SSD. Instead of reading them all from one drive, the expert data are cut into two byte ranges. One range is stored on the Mac's internal SSD and the other on an external NVMe SSD, sized in proportion to what each drive actually delivers, and both are read in parallel. The model is not changed: the split is only a storage layout, and in the V4.1 runs the output was bit-identical (SHA-256) to the unsplit run.

**Test machine.** Mac Studio with an M4 Max and 64 GiB of unified memory; internal Apple SSD (512 GB class); external Samsung 990 PRO 2 TB in a Thunderbolt enclosure. macOS reports that link at 40 Gb/s, with the SSD running at PCIe 8 GT/s x4. When both drives were read at the same time, the internal drive delivered about 5.3 GB/s and the external drive about 3.7 GB/s.

**Split ratio.** 59% internal / 41% external in these tests. This ratio fits this machine's two drives and their connections; it is not a universal constant. Other machines should measure their own drives during real decoding (per-part read bytes and latency under the actual model, not a stand-alone SSD benchmark) and set the ratio accordingly. On this machine, three 68:32 layouts (three runs each, same prompt) averaged 4.57, 4.70 and 4.83 tok/s, against 5.27 tok/s for 59:41.

**DeepSeek V4.1 Flash.** Measured with an experimental SSD-streaming runtime that is not included in this repository (36 GiB expert cache, lookahead 2, 12 threads, 35-token prompt + 200 generated tokens):

| Setup | Runs (tok/s) | Mean |
|---|---|---:|
| One drive (unsplit), 2026-09-26 | 4.573, 4.558, 4.467 | 4.53 |
| 59:41 split, 2026-09-26 | 5.761, 5.089, 5.067 | 5.31 (+17%) |
| 59:41 split, 2026-09-30, different fixed prompt | 5.273, 5.261, 5.286 | 5.27 |

All split reads went through the split path (100%, no fallbacks). The 2026-09-30 row is not a before/after pair: it uses another prompt, and the same layout gave 3.54–3.61 tok/s on a third prompt. For GLM-5.3 Full, the matched one-drive and two-drive check in the table above measured 5.58 and 6.78 tok/s (one run per cell).

The only matched before/after pair is the 2026-09-26 test (+17%). After it, the expert files were repacked so that all 40 expert layers are split, and a matched one-drive comparison has not been repeated on that layout. On 2026-09-30 the current layout measured 3.54–3.61 tok/s on one prompt and 5.27 on another, and I have not yet found why they differ so much, so the size of the gain on the current layout is still open. These numbers come from one machine, short prompts, and two or three runs per setting; they do not predict other machines, prompts, or models.

## Why test another drive?

The present setup already uses two storage devices: the internal SSD and one external NVMe SSD. A device-side replay of the fragmented read pattern measured about **7.4–7.7 GB/s**. The exact decode path was estimated to use about **84–88%** of that measured range, so the two-device setup is close to its measured I/O ceiling. The external SSD sits behind a 40 Gb/s Thunderbolt link that is slower than the drive itself, and the Mac has three unused Thunderbolt 5 ports, so a faster enclosure or a third drive is the next thing to measure. Either could raise the ceiling, while shared connections, scheduling, or other runtime costs may limit the end-to-end gain. Sponsorship funds this controlled measurement and its public report; no speed result is promised.

## What sponsorship supports

1. A third NVMe SSD and, if needed, a compatible enclosure and cable.
2. Controlled one-, two-, and three-device comparisons, with cache effects separated from physical device reads.
3. Public benchmark notes, setup details, and a summary of the result—including negative results.
4. Research tooling. I would like to use a higher-tier AI subscription plan for experiment design, code review and analysis in this work, but I do not currently have the budget for it.

I will add a concrete funding target after selecting the test hardware and checking its current price. Sponsorship does not buy a particular result, private access, or influence over the measurements.

→ **[Support the experiments through GitHub Sponsors](https://github.com/sponsors/jun3982002-droid)**

## Reproducibility and scope

Before treating a number as comparable, check its date, model quantization, prompt, token count, run count, decode/prefill phase, and approximation mode. Older and newer runs can use different prompts or storage settings. These are selected project-reported snapshots.

This is a single-machine research project. Hardware, model quantization, short evaluation texts, and a small set of prompt checks limit what can be concluded. Higher SSD bandwidth may not improve end-to-end inference if another part of the runtime becomes the bottleneck.

This repository publishes experiment summaries, the 2026-09-28 evaluation question sets ([results/eval-sets/](results/eval-sets/)), a single-drive runtime patch, and a consolidated two-drive fast5 patch with a sidecar builder, verifier, and setup recipe ([runtime/](runtime/)). It does not include model weights, generated sidecars/manifests, or the experiment launcher. Both patches target a pinned upstream revision and include their setup, checks, and limitations.

### Credits

The runtime is based on [DS4 by antirez](https://github.com/antirez/ds4). This project reports its own experimental changes and measurements separately from upstream DS4 results. The model is GLM-5.3 Full in IQ2_XXS quantization. This page does not distribute model weights.
