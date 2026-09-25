# CPAP Appeal Checker

**Not medical or legal advice. Not a clinical decision tool. Uses public Medicare policy and
synthetic data only. Coverage rules change — always check the current LCD. Outputs may be wrong
and must be reviewed by a qualified human.**

**A human reviews and sends every letter. This project files nothing.**

---

Reads a fake insurance denial letter and a fake patient record, checks them against Medicare's
real CPAP coverage policy (LCD L33718), and drafts an appeal. For each rule in the policy it says
**met**, **unmet**, or **not enough evidence** — and every met/unmet has to come with a quote that
actually exists in the policy. Quotes are checked by string matching, not by another model. If a
quote can't be found, the answer is thrown out and becomes "not enough evidence."

The point isn't the letter. It's measuring how often it finds the right rule, how often the
quotes are real, how often it invents a clause, and whether accuracy improves when it's allowed
to say "I don't know."

## Results

Six configurations, each run over the same 40 synthetic cases — 239 criterion decisions per
configuration, 1,434 in total. Model: `gemini-3.1-flash-lite`. Policy: **L33718 revision R11**,
effective 01/01/2024, downloaded 2026-09-22.

| # | What changed | Found right rule | F1 | Quotes real | Made up | Abstained | Acc. when answered |
|---|---|---|---|---|---|---|---|
| 0 | whole policy, no retrieval | — | 0.86 [0.77, 0.92] | 0.87 [0.82, 0.91] | 0.05 [0.03, 0.09] | 0.05 [0.03, 0.09] | 0.96 [0.93, 0.98] |
| 1 | naive: fixed chunks, embeddings only | 0.34 [0.28, 0.40] | 0.81 [0.72, 0.89] | 0.89 [0.84, 0.92] | 0.02 [0.01, 0.04] | 0.10 [0.07, 0.15] | 0.96 [0.93, 0.98] |
| 2 | + chunk by rule | 0.65 [0.59, 0.71] | 0.91 [0.82, 0.97] | 0.95 [0.91, 0.97] | 0.02 [0.01, 0.05] | 0.05 [0.03, 0.08] | 0.96 [0.93, 0.98] |
| 3 | + BM25 | 0.67 [0.61, 0.73] | 0.79 [0.72, 0.85] | 0.82 [0.77, 0.87] | 0.00 [0.00, 0.02] | 0.09 [0.06, 0.14] | 0.94 [0.91, 0.97] |
| 4 | + reranker | 0.97 [0.94, 0.99] | 0.90 [0.82, 0.95] | 0.97 [0.94, 0.98] | 0.00 [0.00, 0.02] | 0.05 [0.03, 0.08] | 0.95 [0.92, 0.97] |
| 5 | + quote check + abstain | 0.97 [0.94, 0.99] | 0.90 [0.82, 0.95] | 0.97 [0.94, 0.98] | 0.00 [0.00, 0.02] | 0.05 [0.03, 0.08] | 0.95 [0.92, 0.97] |

Wilson intervals for the proportions. The F1 intervals come from a bootstrap that resamples the
40 **cases**, not the 239 decisions: every decision for one patient comes from the same retrieved
context and the same single model call, so treating them as independent observations would report
intervals far narrower than the evidence supports.

One column needs its name explained. **"Abstained" means the model predicted
`insufficient_evidence`**, which in this project is a real label with its own gold answers — not a
refusal. "Acc. when answered" is accuracy on the decisions where it committed to met or unmet.

### What these numbers support

**Retrieval improves monotonically and the gaps are real:** 0.34 → 0.65 → 0.67 → 0.97, with
non-overlapping confidence intervals. Chunking the policy one rule per chunk is worth 31 points
over fixed-size chunks, and the cross-encoder reranker is worth another 30 on top of that. This
is the one result in the table I would defend without hedging.

**Fabricated citations are eliminated by retrieval quality, not by the verifier.** Rows 4 and 5
invent zero clauses across 239 decisions. Row 0 — the whole policy in context — invents 13.

### What these numbers do not support

**The F1 column cannot rank the configurations.** Every interval overlaps every other one. Rows
2, 4 and 5 are statistically indistinguishable, and row 0, which does no retrieval at all, sits
inside all of them. With 40 cases that column is not precise enough to order anything, and I am
not going to pretend otherwise.

### Three things I did not expect

**BM25 makes it worse.** Row 3 has the lowest F1 in the table (0.79) and the worst quote rate
(0.82) despite retrieving slightly better than row 2. Lexical matching surfaces passages that
share vocabulary with the query but state a different rule.

**Abstention never fires.** In row 5, zero decisions out of 239 were changed by the quote check.
Once retrieval and the prompt were fixed there was nothing left to catch, so row 5 is identical
to row 4 on every metric. The abstention machinery earned its place against an earlier, broken
version of this pipeline — not against this one.

**Every fabricated citation was attached to a correct answer.** Across all six configurations, 23
decisions cited a quote that appears in no policy and in no patient record. All 23 had the right
label. Decisions where the model honestly returned an empty quote were only 69% correct. So quote
verification guarantees that every citation shown to a human is real — it does **not** identify
wrong answers. On this data the unverifiable answers were disproportionately the right ones, and
abstaining on them would have discarded 23 correct decisions to remove none that were incorrect.

The model's own confidence is no help either: mean 0.995, and the gap between its confidence when
right and when wrong is **+0.009**. It is recorded as an exploratory field and should not be used
to rank, threshold, or abstain.

### Where the errors are

Per-criterion accuracy is near-ceiling — `swo` 40/40, `device_instruction` 31/31,
`initial_evaluation` 31/31, `reeval_window` 7/7. Accuracy by case bucket: adversarial 1.00,
clear_unmet 1.00, met_5_14 1.00, clear_met 0.96, borderline 0.94, **insufficient 0.87**. The
hardest bucket is the one where the record is silent, which is the distinction this project
exists to test.

The weakest rule is `short_study_events` at 1/3: on a study under two hours the event minimum
drops from 30 to 10 when a qualifying symptom is documented, and the model applied the higher
threshold and missed the exception.

**One disagreement worth naming.** B1 and B2 are alternative routes to the same coverage
decision. My oracle scores each as written, so a patient with AHI 26 has B1 met and B2 unmet —
B2's 5–14 range simply is not satisfied. The model initially read the same situation as "not
applicable" and returned `insufficient_evidence`. Roughly half of all `unmet` gold labels are
this pattern. `05_results.ipynb` reports the metrics both ways; excluding those decisions leaves
row 4 at 0.90, so the disagreement is not propping up the headline number.

## Running it

```bash
pip install -r requirements.txt
export GEMINI_API_KEY=...        # or put it in Colab secrets
jupyter lab
```

Notebooks run in order, 00 through 05. They pass data to each other through files in `data/`.

`04_experiments.ipynb` has `RUN_FULL = False` by default, which runs four smoke cases (~20 API
calls) instead of all forty (~200). It also has a retrieval-only preflight cell that reports
target-rule coverage before any API call is made — run that first. Responses are cached on disk by
prompt hash, so a repeated run costs nothing, but changing the system prompt invalidates the cache.

## Limitations

- Labels are **made up by me, not annotated by clinicians.** Constructed cases are cleaner than
  real ones, where charts contradict themselves in ways my generator doesn't reproduce.
- **Every case is templated.** Each field is phrased from a bank of three or four variants, so a
  model could be matching phrasing rather than reading the record and nothing here would reveal
  it. PLAN.md called for rewriting ten cases by hand as a control; that was not done, so these
  accuracy numbers should be read as an upper bound.
- Synthetic patients only. No real patient data at any point.
- n = 40. Differences between rows are suggestive, not proof — and as noted above, the F1 column
  cannot separate the configurations at all.
- Criterion selection is **given, not predicted**: the pipeline is told which rules apply to each
  case, using the same phase and device gates the oracle uses. Gold *labels* never enter the
  prompt. What is measured is adjudication, not triage.
- One policy, one model, one revision. Says nothing about how this behaves anywhere else.
- No claim that this is clinically valid, legally sufficient, or that it reduces denials.

## Data

Medicare documents only (public domain), downloaded by hand on 2026-09-22. No scraping. No
commercial insurer policies. No CPT codes anywhere — those are AMA-licensed; this uses HCPCS,
which is free. See `data/SOURCES.md` for the document list and revision numbers.

Code is MIT licensed.
