# 2026-09-28: Three-model comparison on a 64 GB Mac, and a rare derailment in approximate modes

This note covers one night of experiments on an Apple M4 Max (64 GiB unified memory). It records:

1. A small head-to-head test of three locally run models on the same questions.
2. A rare, reproducible "derailment" in one of the experimental approximate GLM modes.
3. A fix for that derailment, what it costs, and what it does not cover.

These are dated measurements from specific runs on one machine. They are not general benchmarks, and they do not guarantee quality or speed.

## Setup

| Model | How it was run |
|---|---|
| GLM-5.3 Full, IQ2_XXS (`jevq6s` mode) | A modified [DS4](https://github.com/antirez/ds4) build. It streams routed experts from the internal SSD plus one external NVMe SSD. `jevq6s` is an experimental approximate mode: cache-aware mass coverage 0.6, resident-expert substitution, and next-layer prefetch of up to 2 predicted experts. |
| Qwen3.8 Flash Next, MLX 3.3 bpw | `mlx-serve`, local OpenAI-compatible server. Its speculative decoding (MTP/PLD) was on, and the KV cache was quantized to 4 bits. The launch script describes the model as about 109 GB, which is larger than the machine's RAM. Swap use rose from about 1.5 GB to 4.9 GB when it started. |
| DeepSeek V4.1 Flash | A separate experimental SSD-streaming runtime ("WARP"), local OpenAI-compatible server. The routed experts use that runtime's 3-bit vector-quantized format (VQ3R), with a mixed-precision trunk. We do not link it here. |

Common conditions:
- Each model was served locally on 127.0.0.1 and run alone, one model at a time.
- Thinking/reasoning mode was off for all three. Temperature was 0.
- Each question was run once per model.
- The questions were in Japanese and graded automatically. Code answers were run against prepared tests in a separate Python process, with a 10-second timeout.
- The question sets are in [`eval-sets/`](./eval-sets/).
- The questions, expected answers and grading scripts were written by an AI coding assistant. For the harder set, every expected answer was checked with a reference implementation, and brute-force enumeration confirmed that each logic puzzle has exactly one solution. There is no such record for the easier set.
- A grading bug was found during the first set: a short answer written with separators (for example `M-F-N-P-O`) was not matched. The grader was fixed to ignore separators, and every model's answers were regraded with the same grader.

About `jevq6s` quality and speed, from earlier measurements:
- NLL was +2.06% and +2.03% versus the exact GLM path on two Japanese texts. NLL is one language-modeling metric, not a task-accuracy score.
- Decode speed was 3.59–3.62 tok/s in one session and 3.85–3.86 tok/s in a later one. Both used the CLI with the same mode, the same fixed 95-token Japanese conversation prompt, and 64 generated tokens. We did not find the cause of the gap.

## Result 1: 24 easier questions

The 24 questions were: arithmetic word problems (6), short logic puzzles (5), Japanese language (4), general knowledge (3), JSON output (3), and short Python functions (3). Each asked for the answer only. Answers were capped at 256 tokens (128 for general knowledge, 512 for code).

| Model | Total | Arithmetic | Logic | Japanese | Knowledge | JSON | Code | Avg. time to first token | Avg. total time per question | Median decode speed |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| GLM-5.3 Full `jevq6s` | **24/24** | 6/6 | 5/5 | 4/4 | 3/3 | 3/3 | 3/3 | 13.3 s | 22.2 s | ~4.1 tok/s |
| Qwen3.8 Flash Next | 22/24 | 6/6 | 3/5 | 4/4 | 3/3 | 3/3 | 3/3 | 0.29 s | 0.45 s | ~98 tok/s |
| DeepSeek V4.1 Flash | 19/24 (see note) | 3/6 | 5/5 | 2/4 | 3/3 | 3/3 | 3/3 | 8.6 s | 17.3 s | ~5.7 tok/s |

Time to first token includes reading (prefilling) the prompt, which is most of the wait for GLM.

Notes:
- **DeepSeek, one logic question:** the answer was cut off at the token limit before giving a final answer. It had started reasoning at length in Chinese although the question was in Japanese. The grader counted it correct because the expected number appeared mid-reasoning. Counting it as incorrect gives **18/24**.
- **DeepSeek's arithmetic misses:** all three were bare numeric answers with no working (346, 12 and 2500 instead of 760, 15 and 1900).
- **DeepSeek's Japanese misses:**
  - Asked for the honorific form of 言う ("to say"), it gave the humble form 申し上げる instead of おっしゃる.
  - Asked for the humble form of 見る ("to see"), it answered with two U+FFFD replacement characters followed by 見する. The expected answer is 拝見する, so the character 拝 was lost.
  - Re-sending that question alone, without streaming, gave the same output. The character is lost, not replaced by a wrong word, so this looks like a text-decoding problem rather than a wrong answer. We did not find the cause. It was graded as incorrect.
- **Qwen's misses:** two logic puzzles. One was a wrong choice. In the other (a letter-shift cipher), it used the right letters in the wrong order.
- **Other GLM modes, for reference (not part of the main table):** `fast4s` scored 23/24 (one copying slip in a final answer after correct reasoning), and `exact` scored 24/24.

## Result 2: 20 harder questions

The 20 questions were: multi-step arithmetic (5), logic puzzles with 4–6 constraints (5), longer Python functions with at least five tests each (5), Japanese (3), and reading comprehension over 400–600 characters (2).
- Arithmetic, logic and reading questions asked the model to end with a line of the form `答え: <answer>` (答え means "answer"). Only that final line was graded.
- Answers were capped at 1024 tokens (512 for Japanese, 1536 for code). A truncated answer counts as incorrect.

| Model | Total | Arithmetic | Logic | Code | Japanese | Reading | Truncated | Avg. time to first token | Avg. total time per question |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| Qwen3.8 Flash Next | **20/20** | 5/5 | 5/5 | 5/5 | 3/3 | 2/2 | 0 | 0.6 s | 6.1 s |
| GLM-5.3 Full `jevq6s` | 18/20 | 5/5 | 4/5 | 4/5 | 3/3 | 2/2 | 0 | 25.9 s | 110.3 s |
| DeepSeek V4.1 Flash | 13/20 | 4/5 | 2/5 | 3/5 | 2/3 | 2/2 | 3 | 26.2 s | 134.5 s |

Notes:
- **GLM, code miss:** the expression parser called a helper before defining it inside the same function. This raised `UnboundLocalError`, although the parser design itself was correct.
- **GLM, logic miss:** this was not a reasoning error. See the next section.
- **DeepSeek, code misses:** one expression evaluator ignored operator precedence (`1+2*3` gave 9). One date-difference function ignored whole years.
- **DeepSeek, truncations:** all three were long reasoning in Japanese that hit the limit before a final answer line.
  - Arithmetic: it wrote `答え: 31` partway through (the correct answer is 180), then kept checking.
  - One logic puzzle: it went through the same cases several times.
  - Another logic puzzle: it wrongly concluded that the answer was not unique, and was still re-checking.
- **DeepSeek, formatting:** one logic answer reasoned in Chinese and reached the correct answer. Its last line used the Chinese word `答案:` ("answer") instead of the requested `答え:`, so the grader marked it incorrect.
- **DeepSeek, garbling:** the same replacement-character problem appeared twice more.
  - In the Japanese question about the idiom 汚名挽回, the character 汚 was garbled every time it appeared. 汚名挽回 is a common misuse; the standard forms are 汚名返上 ("clearing one's name") and 名誉挽回. The explanation was otherwise right, but the grader looks for the expected phrase, so this was marked incorrect.
  - In one truncated logic answer, 解釈 ("interpretation") came out as 解 plus two replacement characters.
  - No garbling was seen in the GLM or Qwen answers of this comparison. GLM did show it twice in the later fix-verification runs (one replacement character each, in `fast4s` and in `jevq6s` with N=16). So this symptom is not unique to DeepSeek's runtime.

**Reading these results:** the question sets are small (24 and 20), so a difference of one or two questions should be read as a tie. Across both sets, Qwen3.8 Flash Next and GLM `jevq6s` scored higher than DeepSeek V4.1 Flash as run here. Qwen answered roughly 20–50× faster per question. All three were quantized builds, and GLM and DeepSeek ran on experimental SSD-streaming runtimes, so these results say nothing about the full-precision models.

## Result 3: a rare, reproducible derailment in an approximate GLM mode

### What happened
On one harder logic puzzle (four people and four pets), GLM `jevq6s` did not answer the question at all. From the second generated token onward, it wrote an unrelated 984-token article about causes of ear pain, then stopped normally. The prompt was read correctly (189 tokens), and no conversation state was reused.

### Reproduction and root cause

| Test | Result on the pet puzzle |
|---|---|
| Server, fresh start: `jevq6s`, the puzzle alone | Correct |
| Server, fresh start: `jevq6s`, the preceding puzzle and then this one | Correct |
| **Server, fresh start: `jevq6s`, the same 10 questions in the original order** | **Derails again. All 10 outputs were byte-identical to the original run.** |
| Server, fresh start: `exact` (no approximation), the same 10 questions in the same order | Correct. The output was identical to asking the puzzle alone. |
| CLI (no server), the puzzle alone: `jevq6s` / `exact` / `fast4s` / older `jevq` | All start reasoning correctly. The CLI uses a slightly different prompt template (195 tokens instead of 189). |

**Cause:** the approximate modes decide which experts to skip or substitute partly from what is currently in the expert cache, and the cache contents depend on the earlier requests. After this particular sequence of 9 questions, the approximation made a bad choice at the second generated token, and generation never recovered. The exact path does not depend on cache contents, and in our test it did not derail.

The average quality metric (NLL) cannot show a failure like this, because it hides rare catastrophic outputs. We saw one such derailment in 44 graded answers from `jevq6s`.

### Fix: exact decoding for the first N decode steps

We added a switch, `DS4_GLM_APPROX_WARMUP_TOKENS=N`. For the first N decode steps of every request, approximation is turned off: no skipping or substitution, while prefetch stays on. After that, the mode's normal approximation resumes.
- Prompt processing (prefill) was already exact, and the first generated token comes from it. So a 64-token answer has 63 decode steps, and N=16 makes generated tokens 2 through 17 exact.
- With the switch unset, output is byte-identical to the build before the change. We checked this by SHA-256 on the CLI, with a fixed 95-token Japanese conversation prompt and 64 generated tokens, for `jevq6s` and `fast4s`. The `exact` output of the production build also kept its previous hash.

Results for `jevq6s` on the reproducing 10-question sequence. The speed columns are from the CLI (fixed 95-token Japanese conversation prompt, 64 or 256 generated tokens), averaged over two runs each.

| Warm-up N | Pet puzzle | 10-question score | Decode, 64 tokens | Decode, 256 tokens |
|---:|---|---:|---:|---:|
| 0 (off) | **Derails** | 9/10 | 3.86 tok/s | 3.95 tok/s |
| 8 | Correct | 10/10 | 3.53 (−8.5%) | 3.88 (−1.9%) |
| 16 | Correct | 10/10 | 3.28 (−15%) | 3.78 (−4.4%) |
| 32 | Correct | 10/10 | 2.77 (−28%) | 3.61 (−8.7%) |

- NLL was measured only at N=32. It changed from 1.363123 to 1.368080 (+0.36%, one text).
- The full 20-question hard set was re-run only at N=8. It scored 18/20 again, but with a different mix.
  - The pet puzzle was fixed.
  - One reading question was newly missed. The working reached the correct total of 85,450, but the final line said 84500.
  - No other derailments were seen.
  - It is not established whether that miss comes from N=8 or is ordinary variation between runs.

**Adopted default: N=16.** It is the default in our launch scripts for the approximate modes that run on the patched build. One experimental mode, `fast4p`, runs on a separate binary without this change and is not covered. The exact path is unaffected.
- **Why 16:** N=8 was the smallest value tested, and it fixed the reproduced case. We have not ruled out that the new reading miss at N=8 was caused by the smaller window. N=16 is a middle choice between cost and that risk.
- **What N=16 was checked on:** the 10-question sequence (10/10), a 64-token output on the production build, and the speed table above. It was not run on the full 20-question set, and its NLL was not measured.
- **Cost:** it is concentrated in short answers.
  - On the production build at 64 tokens (one run each), `jevq6s` went from 3.85 to 3.28 tok/s (−15%).
  - `fast4s` went from 5.54 to 3.88 tok/s (−30%). Its cost at 256 tokens was not measured.
- **Why `fast4s` also gets it:** `fast4s` did not derail on the 10-question sequence. But it relies on the same cache-dependent skipping and substitution, and it approximates more heavily (NLL +6.0% versus the exact path, against +2.06% for `jevq6s`). One sequence without a derailment does not show that it cannot happen. In the binary, the warm-up is off when `DS4_GLM_APPROX_WARMUP_TOKENS` is unset or 0. Our launch scripts set it to 16 unless told otherwise, so turning it off there needs the scripts' own switch.

**What this fix does not cover:** it only protects the start of an answer. A derailment that begins later in a long answer would not be prevented. It also has only been shown to fix one reproduced case.

## Limitations

- One machine, one session. The question sets are small and in Japanese, and each question was run once per model.
- The models were quantized, and GLM and DeepSeek ran on experimental runtimes. Speeds include each runtime's own behavior, such as SSD streaming for GLM and DeepSeek.
- Approximate GLM modes can give different outputs for the same prompt depending on earlier requests (cache state). In our tests, the exact mode did not.
- The runtime changes are not yet published in this repository.
