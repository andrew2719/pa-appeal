# CPAP Appeal Checker

**Status: in progress.** Working through `PLAN.md`. Results table below is not filled in yet.

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

| # | What changed | Found right rule | F1 | Quotes real | Made up | Abstained | Acc. when answered |
|---|---|---|---|---|---|---|---|
| 0 | whole policy, no retrieval | — | | | | | |
| 1 | naive: fixed chunks, embeddings only | | | | | | |
| 2 | + chunk by rule | | | | | | |
| 3 | + BM25 | | | | | | |
| 4 | + reranker | | | | | | |
| 5 | + quote check + abstain | | | | | | |

Every number gets a 95% confidence interval. n = 40, so the intervals are wide.

## Running it

```bash
pip install -r requirements.txt
export GEMINI_API_KEY=...        # or put it in Colab secrets
jupyter lab
```

Notebooks run in order, 00 through 05. They pass data to each other through files in `data/`.

## Limitations

- Labels are **made up by me, not annotated by clinicians.** Constructed cases are cleaner than
  real ones, where charts contradict themselves in ways my generator doesn't reproduce.
- Synthetic patients only. No real patient data at any point.
- n = 40. Differences between rows are suggestive, not proof.
- One policy. Says nothing about how this behaves anywhere else.
- No claim that this is clinically valid, legally sufficient, or that it reduces denials.

## Data

Medicare documents only (public domain), downloaded by hand. No scraping. No commercial insurer
policies. No CPT codes anywhere — those are AMA-licensed; this uses HCPCS, which is free.

Code is MIT licensed.
# pa-appeal
