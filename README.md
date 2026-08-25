# Evaluating Automatic Metrics and LLM-as-Judge for Clinical Text Summarization

Code, run logs, and aggregate results for my BSc thesis (CSE, Rajshahi University of Engineering & Technology).

The question behind this repository is a simple one that turns out to be awkward to answer: **if you use ROUGE, BLEU, or BERTScore to evaluate clinical summaries, are you measuring anything useful?**

The usual way people check is to correlate metric scores against some quality judgement across a pile of summaries from different systems. That gives reassuring-looking correlations. But it quietly answers a different question than the one most researchers actually care about. Telling a weak model apart from a strong one is easy. Ranking two summaries from the *same* model — which is what you do when you are tuning a prompt, picking a checkpoint, or deciding whether a change helped — is much harder, and the pooled correlation hides exactly that.

So this project runs both analyses on the same data and reports the gap.

---

## What was actually run

Brief Hospital Course summaries were generated for **300 de-identified discharge-note cases** from MIMIC-IV-BHC under three conditions, giving **900 summaries** in total:

| Condition | Model | Why it's there |
|---|---|---|
| `weak_bart` | `facebook/bart-large-cnn` | deliberately weak, off-domain baseline |
| `zero_shot` | `Qwen/Qwen2.5-7B-Instruct`, instruction only | stronger condition |
| `few_shot` | same generator + 2 held-out demonstrations | in-context adapted condition |

Every case appears in all three conditions, so the design is fully paired and the statistics are clustered on the source case rather than on the row.

Each summary was scored by five automatic metrics (ROUGE-1/2/L, sentence-level SacreBLEU, baseline-rescaled BERTScore) and by two LLM judges — `Qwen2.5-14B-Instruct` as primary and `Qwen2.5-7B-Instruct` as a same-family capacity sensitivity check. The judge rubric has three dimensions (consistency, completeness, coherence), each scored 1–5, and the mean is used as the headline judge score.

Because an LLM judge is not ground truth, the same judge, same rubric, and same settings were also run against **SummEval** (50 complete articles × 16 system summaries = 800 rows), where real human expert ratings exist. That gives an external, human-anchored check on the instrument itself.

---

## Headline findings

**1. Pooled agreement overstates fine-grained agreement, and does so unevenly.**

| Metric | Pooled ρ | Condition-adjusted ρ | % of pooled lost |
|---|---:|---:|---:|
| ROUGE-1 | 0.355 | 0.150 | 57.6% |
| ROUGE-2 | 0.398 | 0.253 | 36.5% |
| ROUGE-L | 0.380 | 0.204 | 46.4% |
| BLEU | 0.410 | 0.200 | 51.3% |
| **BERTScore F1** | **0.371** | **0.343** | **7.7%** |

All ten intervals exclude zero (2,000 case-clustered bootstrap replicates). The ranking flips: BLEU leads when systems are pooled, BERTScore leads once system identity is controlled. Adding length ratio as a second control does not change this.

**2. BERTScore is the strongest metric inside every condition.**

Weak BART 0.440 [0.343, 0.538], zero-shot 0.223 [0.114, 0.328], few-shot 0.346 [0.243, 0.440]. Its lead over each lexical metric survives a paired bootstrap contrast (vs ROUGE-1 +0.192, p = 0.001; vs BLEU +0.143, p = 0.001; vs ROUGE-L +0.139, p = 0.001; vs ROUGE-2 +0.090, p = 0.004).

**3. On SummEval, the judge beats every metric against human ratings.**

Dataset-level: judge 0.443 [0.381, 0.506] vs BERTScore 0.286 and ROUGE-1 0.285. Summary-level: judge 0.389 vs BERTScore 0.196. Every paired difference has a bootstrap p of 0.001 with an interval excluding zero. The 7B judge lands between the 14B judge and the metrics (0.380 dataset-level), so capacity matters but the ordering does not depend on it.

**4. High metric scores were *not* systematically misleading.**

Across all 15 condition × metric combinations, top-quartile metric scores contained *fewer* low-judge summaries than the condition base rate — never more. The problem with these metrics here is reduced sensitivity within a narrow quality band, not a tendency to reward bad summaries. That is a meaningfully different failure mode from the one usually assumed, and I think it's the more honest reading.

**5. Few-shot was not detectably better than zero-shot.** Mean difference 0.035, 95% CI [−0.005, 0.076], paired Cohen's *d* = 0.10, Wilcoxon *p* = 0.11. Both clearly beat the weak baseline; the two-demonstration prompt did not clearly beat plain instructions.

Inter-judge agreement (14B vs 7B, 900 summaries): ρ = 0.606 [0.558, 0.653].

Full numbers for all of the above are in `Results/public_results/`.

---

## ⚠️ Data access — please read before cloning

MIMIC-IV-BHC is derived from MIMIC-IV, which is a **credentialed PhysioNet resource, not open data**.

**Nothing in this repository contains clinical text.** No source notes, no reference summaries, no generated summaries, no per-row clinical scores. What is published here is aggregate only — correlations, confidence intervals, and descriptive statistics. The run logs and notebook outputs were checked for text leakage before publication.

To reproduce the clinical half of the pipeline you need your own PhysioNet credentialing (including the CITI training) and your own copy of the dataset. Point `data_path` in the config at wherever you mount it. The SummEval half uses public data and runs without any of that.

Files that the pipeline produces but that must **never** leave a credentialed environment:

```
selected_cases.csv         generated_summaries.csv      metrics_long.csv
clinical_results_final.csv judge_raw_clinical.jsonl     gen_checkpoint.jsonl
judge_audit_sheet.csv
```

If you rerun this, put those in `.gitignore` before your first commit. Stage 5 writes a separate `public_results/` folder containing only aggregate tables, precisely so results can be shared without the text.

---

## Repository layout

```
Notebooks/
  thesis-a-generation-stage1.ipynb     Stage 1
  thesis-b-judging-stage2.ipynb        Stage 2
  thesis-b-judging-stage3-4.ipynb      Stages 3 and 4
  thesis-c-analysis-stage5-6.ipynb     Stages 5 and 6
Logs/
  thesis-a-generation-stage1.log       raw Kaggle console log, one per session
  thesis-b-judging-stage2.log
  thesis-b-judging-stage3-4.log
  thesis-c-analysis-stage5-6.log
Results/
  public_results/                      the 14 result tables, no clinical text
    index.json
    table1_descriptives.csv  ...  table14_judge_vs_metrics_on_humans.csv
README.md
```

Notebook and log filenames are kept exactly as Kaggle produced them, so each log pairs with its notebook by name. The `a` / `b` / `c` prefixes are the session grouping, not the stage number — session `b` covers Stages 2, 3, and 4 across two notebooks, for the reason explained below.

All four notebooks contain the **same** single-file pipeline. Only one line differs between them: the `STAGES` set at the top. They are kept separate because Kaggle attaches logs and outputs to notebooks rather than to scripts, and the per-session logs are part of the reproducibility record.

`index.json` lists every table with its row count, columns, and notes, so you can find the right file without opening all fourteen.

---

## The pipeline

Six stages. Each writes to `/kaggle/working`, checkpoints every row, and reads the previous stage's outputs.

| Stage | What it does | Device | Measured wall-clock |
|---|---|---|---|
| 1 | sampling + generation (900 summaries) | GPU | **5.0 h** |
| 2 | ROUGE, BLEU, BERTScore | GPU | ~4 min |
| 3 | both judges on 900 clinical summaries | GPU | ~2.7 h |
| 4 | SummEval judging + metrics (800 rows) | GPU + internet | ~2.0 h |
| 5 | analysis, 14 tables, 3 figures | CPU | ~6 min |
| 6 | independent integrity re-check | CPU | ~1 min |

Total is roughly **10 hours**, which is why it cannot run in one Kaggle session.

Stage 1's own generation loop took 296.9 minutes; the figures above are full session times from the logs, so they include package installs and model downloads. The docstring inside the pipeline carries slightly more pessimistic pre-run estimates — the table here is what actually happened.

### Running it on Kaggle

Edit one line at the top of the notebook:

```python
STAGES = {1}        # thesis-a-generation-stage1
STAGES = {2}        # thesis-b-judging-stage2
STAGES = {3, 4}     # thesis-b-judging-stage3-4
STAGES = {5, 6}     # thesis-c-analysis-stage5-6
```

Then *Save & Run All*. For every session after the first: right panel → **Add Input → Notebook Output** → pick the previous session's run. `PREV_INPUT = None` auto-detects the mount; you can also paste the path manually. Stage startup copies the previous outputs into `/kaggle/working` so every stage reads and writes one directory.

**This split is not optional, and not only about the 12-hour cap.** Running the whole thing in a Kaggle *draft* session works; running it in one *interactive* session did not. Two separate things go wrong:

- **GPU memory.** Stage 1 holds a 7B generator and BART; Stage 3 needs a 4-bit 14B judge across both T4s. Freeing one model inside a live session is not reliable enough to hand the next stage a clean device.
- **Package version conflicts.** Stage 2 upgrades `bert-score` and its dependencies. Trying to load Qwen afterwards in the same session crashed with:

  ```
  StrictDataclassDefinitionError: Class 'Qwen2Config' must be a dataclass before applying @strict.
  ```

  which is a `transformers` / `huggingface_hub` mismatch introduced by the upgrade, not a bug in this code. A fresh session fixes it. That is why Stage 2 and Stage 3 are listed as separate sessions above even though Stage 2 only takes four minutes. The full traceback is in `Logs/thesis-b-judging-stage2.log`.

If a session dies partway, just rerun it. Checkpointing is per-row and keyed on a configuration fingerprint, so completed rows are reused and nothing is silently mixed across configurations.

### Running it elsewhere

The pipeline falls back to `./work` if `/kaggle/working` doesn't exist, so it runs locally given roughly 16 GB of VRAM for the 4-bit 14B judge and a mounted copy of the dataset. It has only been tested on Kaggle's 2×T4 setup.

---

## Configuration

Everything lives in one frozen dataclass near the top of the file — seed 42, 300 evaluation cases, 2 demonstrations, 4,000-token input limit for Qwen, 1,024 for BART, 512 max new tokens, deterministic decoding, 2,000 bootstrap replicates, top-quartile threshold, judge cutoff 3.0 for the divergence analysis.

Two details worth knowing if you modify it:

- **Split fingerprints.** Generation settings and judging settings are hashed separately. Changing the rubric or the SummEval sample therefore does not invalidate summaries that already cost five hours to produce. This was learned the hard way.
- **The rubric text is part of the instrument** and enters the judging fingerprint. Edit a word of it and you get a fresh set of checkpoint keys instead of a quietly contaminated mixture of old and new scores.

---

## Reproducibility

Stage 6 is an independent re-check that shares no helper code with Stage 5. It verifies row counts and the paired structure, confirms no demonstration leaked into the evaluation set, checks every metric is finite and on its documented scale, confirms each judge mean equals the mean of its three dimensions, enforces a parsing-coverage floor per judge, recomputes the headline coefficients from scratch, and confirms one consistent generation fingerprint across Stages 2–6. The final run ends with `ALL INTEGRITY CHECKS PASSED`; that output is in `Logs/thesis-c-analysis-stage5-6.log`.

Scale conventions, since these get mixed up constantly: ROUGE is 0–1, SacreBLEU sentence scores are 0–100, and **baseline-rescaled BERTScore can legitimately be negative** — the condition means here are −0.034, −0.021, and −0.014. Raw magnitudes are not comparable across metric families; only the rank correlations are.

---

## What this does not show

I would rather state these plainly than have someone else find them.

- **No clinician ever read these summaries.** The LLM judge is a proxy that has been externally benchmarked on general-domain human ratings. It is not clinical validation, and nothing here says these summaries are safe to use.
- **The judge is reference-based, not source-grounded.** It compares candidate against reference BHC without seeing the original notes, so what it measures is reference consistency, not full clinical factuality.
- **Generator and primary judge share a model family** (Qwen2.5-7B and Qwen2.5-14B). The 7B judge is a *capacity* sensitivity check, not an independent-family one. A cross-family judge is the obvious next step and is not in this repository.
- **SummEval validation is out-of-domain** — news, not clinical text.
- Within-condition correlations are affected by range restriction. That is the point of the analysis, but it also means these numbers are not portable to a setting with a wider quality spread.
- One dataset, one language, one fixed two-example prompt, one clinical domain.
- Some generated summaries insert demographic details the reference does not support, despite prompt instructions not to. That problem was reduced, not eliminated.
- Bootstrap p-values printed as `0.000` are reported as p < 0.001, because 2,000 replicates cannot resolve anything finer.

---

## Related work this builds on

The pooled-versus-fine-grained problem is not new — Peyrard (2019) and Bhandari et al. (COLING 2020) both show that metric agreement depends heavily on the scoring range being compared, and SummEval (Fabbri et al., TACL 2021) provides the human ratings used here. What this project adds is a controlled, fully paired clinical instance of it, with the *differential* result: under an identical range restriction, BERTScore holds up and the lexical metrics do not.

## Citation

```bibtex
@mastersthesis{firdows2026metrics,
  author = {Nahian Faiza Firdows},
  title  = {Evaluating Automatic Metrics and LLM-as-Judge for Clinical Text Summarization},
  school = {Rajshahi University of Engineering \& Technology},
  type   = {BSc thesis},
  year   = {2026}
}
```

Supervised by Md. Farhan Shakib, Department of Computer Science and Engineering, RUET.

## License

Code is MIT. Result tables are CC BY 4.0. Neither covers MIMIC-IV-derived data, which remains under the PhysioNet data use agreement and is not distributed here.