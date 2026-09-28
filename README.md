# Large MoE Inference on a 64 GB Mac

**Can a 64 GB Mac run a model whose weights are much larger than memory—and what trade-offs make it practical?**

This project measures SSD-streamed inference for **GLM-5.3 Full, IQ2_XXS quantization** (196.58 GiB model file) on an Apple M4 Max with 64 GiB of unified memory. The experimental runtime is a modified [DS4](https://github.com/antirez/ds4) build. The current storage layout reads from the Mac's internal SSD and one external NVMe SSD.

## Selected results

These are dated measurements from specific experiments, not a general performance guarantee.

| Experiment | Measured result | Conditions and limits |
|---|---|---|
| `jevq6s` decode, 2026-09-28 | 3.62 / 3.59 tok/s across two runs; exact measured about 2.1 tok/s | Experimental approximate mode combining cache-aware MASS, resident-expert substitution, and next-layer prefetch. In separate NLL evaluations, +2.06% vs exact on one Japanese text (959 tokens scored) and +2.03% on another (2,222 tokens scored). These two texts do not establish general response quality. |
| Exact vs. `fast4s` decode, 2026-09-27 | 1.955 vs. 5.555 generated tokens/s (2.84× ratio of the two-run means) | Same fixed Japanese prompt, 64 generated tokens, 2 runs per mode. `fast4s` is approximate; it substitutes already-resident experts for selected cache-missing experts. This is not an apples-to-apples quality comparison. |
| `fast4p` prefill, 2026-09-27 | 4.535 → 5.785 input tokens/s (+27.56%); decode stayed at 5.59 → 5.615 output tokens/s | 95-token input, two paired runs. `fast4p` changes prefill behavior while retaining the `fast4s` decode configuration. |
| Exact-mode memory threshold, 2026-09-28 | 2.135 tokens/s with static weights locked vs. 1.85 tokens/s when they were pageable | Two runs per boundary setting, plus one control run. The normal automatic setting keeps the weights locked; increasing the expert cache past the boundary made things slower. |

### Quality notes

- In a separate `fast4s` evaluation, NLL was **5.98% higher than exact** on one 991-token text (959 tokens scored). This is a change in one language-modeling metric, not “6% quality loss,” accuracy, or a broad measure of usefulness.
- In a separate `fast4p` comparison with `fast4s`, NLL was +0.8128% and +1.0984% on two texts (991 tokens each; 150-token prefix, 841 continuation tokens scored). A few short JSON, arithmetic, and Japanese prompts were also checked. Earlier longer checks included an unfinished Japanese response and an arithmetic answer that was wrong in both modes; later short-answer checks completed, but these small samples do not establish general task quality.
- Exact-mode output hashes matched the reference in the tested cases. That claim applies to those checks only.

## Choose a mode

The experimental launcher already accepts these modes by name on the test machine. This repository does not yet include the launcher or modified runtime, so the table is a guide to measured choices, not a runnable download.

| Mode | Decode speed | NLL difference vs. exact | When to choose |
|---|---:|---:|---|
| `exact` | ~2.1 tok/s | 0 (reference) | Use the reference path when fidelity is the priority. Output hashes matched the reference in tested cases only. |
| `jevq6s` | ~3.6 tok/s | +2.03% / +2.06% on two Japanese texts | Quality-priority starting point; the current recommended preset for these tests. |
| `jevq5s` | ~3.8 tok/s | +3.69% / +3.81% on the same two texts | A little more speed, with a larger measured NLL difference in those texts. |
| `fast4q` | ~4.5 tok/s | +4.5% in the reported evaluation | Middle-speed option. |
| `fast4s` | 5.3–5.5 tok/s | +5.98% on one 991-token text (959 scored) | Choose when decode speed matters more. |
| `fast5` | ~6.7 tok/s | +21.48% on one 991-token text (959 scored) | Highest measured speed in this set; not recommended as the default because of the larger measured NLL difference. |

These are project snapshots from different runs and evaluation conditions, not a single apples-to-apples benchmark. NLL is a language-modeling metric, not a percentage score for answer quality or accuracy; the text evaluations are small. `fast5` is distinct from `fast5s`, a separate experimental mode that was withdrawn after a malformed-output check.

## How the approximation works

The `jevq6s` mode combines cache-aware expert selection, resident-expert substitution, and next-layer prefetch. The `fast4s` mode can skip selected experts that are not in the memory cache, then use a suitable expert that is already resident. These modes reduce some SSD reads but change the computation. `exact` remains available as the reference mode. Approximate output is not described as lossless.

## Why test another drive?

The present setup already uses two storage devices: the internal SSD and one external NVMe SSD. A device-side replay of the fragmented read pattern measured about **7.4–7.7 GB/s**. The exact decode path was estimated to use about **84–88%** of that measured range, so the two-device setup is close to its measured I/O ceiling. A third drive could raise that ceiling, while shared connections, scheduling, or other runtime costs may limit the end-to-end gain. Sponsorship funds this controlled measurement and its public report; no speed result is promised.

## What sponsorship supports

1. A third NVMe SSD and, if needed, a compatible enclosure and cable.
2. Controlled one-, two-, and three-device comparisons, with cache effects separated from physical device reads.
3. Public benchmark notes, setup details, and a summary of the result—including negative results.

I will add a concrete funding target after selecting the test hardware and checking its current price. Sponsorship does not buy a particular result, private access, or influence over the measurements.

→ **[Support the experiments through GitHub Sponsors](https://github.com/sponsors/jun3982002-droid)**

## Reproducibility and scope

Before treating a number as comparable, check its date, model quantization, prompt, token count, run count, decode/prefill phase, and approximation mode. Older and newer runs can use different prompts or storage settings. These are selected project-reported snapshots.

This is a single-machine research project. Hardware, model quantization, short evaluation texts, and a small set of prompt checks limit what can be concluded. Higher SSD bandwidth may not improve end-to-end inference if another part of the runtime becomes the bottleneck.

This repository currently publishes experiment summaries, not the modified runtime source or reproduction scripts. Those materials are being prepared separately. Before publishing them, I will identify the exact upstream revision, review the patch and bundled notices, and remove private paths or prompts from any logs.

### Credits

The runtime is based on [DS4 by antirez](https://github.com/antirez/ds4). This project reports its own experimental changes and measurements separately from upstream DS4 results. The model is GLM-5.3 Full in IQ2_XXS quantization. This page does not distribute model weights.
