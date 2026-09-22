# CPAP Appeal Checker — my build plan

Notebooks only. No `src/` folder, no imports from `.py` files, no config files, no CLI.
Everything happens in cells. Data moves between notebooks as JSON files on disk.

---

## 1. What I'm building, in one paragraph

Medicare has a public rulebook for when it will pay for a CPAP machine. I have a fake patient's
paperwork and a fake denial letter. Go through the rulebook one rule at a time and decide: does
this paperwork prove the patient qualifies? Three possible answers — **met**, **unmet**, or
**not enough evidence**. Every met/unmet has to come with a quote from the rulebook, and I check
that the quote actually exists by searching for it in the text. If the quote isn't really there,
I throw the answer away and call it "not enough evidence."

Then it writes an appeal letter. A human reads and sends it. My code sends nothing.

**The letter is not the point.** The point is measuring: how often does it find the right rule,
how often are the quotes real, how often does it invent a clause, and does accuracy go up when
it's allowed to say "I don't know."

## 2. Data, process, model

**Data — three kinds, only one is real.**

- *The rulebook.* Medicare LCD **L33718**, CPAP for sleep apnea. Real, public, I download it.
- *Decoy rulebooks.* ~8 unrelated Medicare equipment policies (oxygen, wheelchairs, nebulizers…).
  These exist so that finding the right document is not automatic. With one document in the pile,
  "did it find the right one?" is always yes and the number is worthless.
- *Patients.* Completely fake. I write a Python function that invents 40 of them. Because I
  invented them, **I already know the right answer for every one** — that's my answer key. No
  doctors, no labelling, no real records ever.

**Process — for one patient.**

1. Cut the rulebook into ~15 numbered rules.
2. Find the rules that look relevant to this patient.
3. Send those rules + the patient's paperwork to an LLM. Ask for JSON back: for each rule, the
   label, a quote, and one line of reasoning.
4. Take each quote and literally search for it in the rulebook text. `if quote in policy_text`.
5. Quote not found → label becomes "not enough evidence," no matter how confident the model was.
6. Put whatever survived into a letter template.

Step 4 is not AI. It's string matching. That's on purpose — it's the one number in this project
nobody can argue with.

**Model — I train nothing.**

| Piece | What it does | Where it comes from |
|---|---|---|
| Embedding model | turns text into numbers so I can measure "similar" | free download, runs on CPU |
| BM25 | old-school keyword search, catches exact strings like `E0601` | a tiny pip package |
| Reranker | re-sorts the top results more carefully | free download |
| An LLM via API | reads the case and fills in the JSON form | costs a little money |

No fine-tuning. No training loop. No GPU required.

## 3. The rules of the rulebook I'm modelling

This is the actual policy content. Everything else is plumbing.

**To get a CPAP (code E0601) in the first place**, the sleep study has to show either:
- AHI or RDI **≥ 15 per hour, with at least 30 events total**, or
- AHI or RDI **between 5 and 14, with at least 10 events**, *plus* at least one of: excessive
  daytime sleepiness, impaired cognition, mood disorder, insomnia, hypertension, ischemic heart
  disease, history of stroke.

**To keep it past the first 3 months:**
- a re-evaluation **between day 31 and day 91**, and
- **used ≥ 4 hours a night on 70% of nights** across some 30-day stretch in the first 3 months.

**Definitions:** apnea = airflow stops for ≥10 seconds. Hypopnea = ≥10 seconds, ≥30% less
airflow, ≥4% oxygen drop.

**Device codes:** `E0601` = CPAP. `E0470` = bi-level, only covered for sleep apnea if E0601 was
tried and didn't work. `E0471` = bi-level with backup rate, **not** covered when the main
diagnosis is sleep apnea.

Everything here is numbers and counting. That's why this policy and not another one — I can check
answers objectively.

## 4. Hard rules I don't break

- Fake patients only. Never real patient data.
- Only Medicare documents in the repo (public domain). No commercial insurer policies — copyrighted.
- **No CPT codes anywhere.** They're AMA-licensed. CPAP uses HCPCS codes, which are free. Easy.
- Download documents by hand. No scraping.
- Never claim this is clinically valid, legally sufficient, or that it reduces denials.
- Say "a human reviews and sends it" at the top of the README.
- MIT license on the code.

## 5. How I actually work

**Everything in notebooks.** When something works in a cell, I leave it in that cell. I don't
move it to a `.py` file.

**Notebooks talk to each other through files.** Notebook 1 saves `criteria.json`. Notebook 2 reads
it and saves `cases.json`. And so on. No importing.

**One rule I do follow, because the results depend on it:** notebook 04 has to produce all six
rows of the results table in a single top-to-bottom run. Before I record numbers, I hit *Restart
kernel → Run all* and let it run start to finish. If the numbers only appear when I run cells in
some special order, they're not real numbers. This is the one bit of discipline that matters, and
it costs nothing — it's just a habit.

**Folders:**

```
pa-appeal/
├─ README.md
├─ PLAN.md                    ← this file
├─ requirements.txt
├─ data/
│  ├─ policies/               13 .md files (1 real + ~8 decoys + a few extras)
│  ├─ criteria.json           my ~15 rules with exact quotes
│  ├─ cases.json              40 fake patients + answer key
│  └─ results/                one .json per experiment
└─ notebooks/
   ├─ 00_toy.ipynb
   ├─ 01_policy.ipynb
   ├─ 02_cases.ipynb
   ├─ 03_pipeline.ipynb
   ├─ 04_experiments.ipynb
   └─ 05_results.ipynb
```

`pip install` list: `markdownify beautifulsoup4 numpy rank-bm25 sentence-transformers pydantic
rapidfuzz scikit-learn matplotlib tqdm` plus whichever API client I'm using. In Colab, run the
install cell each session.

---

# The notebooks

## 00_toy.ipynb — do this first, it's one afternoon

Before anything else. No retrieval, no chunking, no 40 cases.

1. Download L33718, save it as a text file.
2. Write **three** fake patients by hand, straight into a cell as strings. One obvious yes, one
   obvious no, one where the record just doesn't say.
3. Paste the *whole policy* into a prompt along with one patient. Ask for JSON: for two or three
   rules, give me a label, a quote, and a reason.
4. For each quote it returns: `print(quote in policy_text)`.
5. Look at what happened.

**Why this first:** I need to personally watch the model return a confident quote that isn't in
the document. Right now that's something I've been told. After this it's something I've seen, and
the rest of the project stops feeling arbitrary.

**Bonus:** this is secretly row 0 of my final results table (whole policy in context, no
retrieval), so it isn't throwaway work.

**Done when:** I've seen at least one quote come back `False`.

---

## 01_policy.ipynb — get the documents, write down the rules

Download by hand from Noridian or CGS (easier to save cleanly than cms.gov):

- **L33718** — the one that matters
- A52467, NCD 240.4, NCD 240.4.1, A55426 — supporting documents
- **~8 unrelated equipment policies as decoys** — oxygen, hospital beds, wheelchairs, nebulizers,
  glucose monitors, surgical dressings, infusion pumps, power mobility

Save each as HTML, convert with `markdownify` in a cell, write out to `data/policies/`.

Then the important part: **read L33718 with a pen and write out my rules.** In a cell, build a
list of dicts and save it as `criteria.json`:

```python
criteria = [
    {
        "id": "B1",
        "summary": "AHI/RDI at least 15 with at least 30 events",
        "text": "",          # ← paste the EXACT sentence from L33718, copy-paste, no retyping
        "source": "L33718",
    },
    # ...about 15 of these
]
```

Rules to cover: B1, B2, the symptom list, apnea definition, hypopnea definition, the day 31–91
window, the adherence definition, E0601, the E0470 prior-trial requirement, E0471 not covered for
sleep apnea.

**The `text` field has to be copy-pasted, character for character.** Not retyped, not
paraphrased. This file is doing two jobs: it's how I cut the policy into pieces, *and* it's the
answer key I check retrieval against. If the text is slightly off, every number downstream is
quietly wrong and I won't be able to tell.

Check it in a cell:

```python
policy = open("data/policies/L33718.md").read()
for c in criteria:
    print(c["id"], c["text"] in policy)
```

Everything prints `True` or I fix it before moving on.

**Done when:** ~13 markdown files on disk, `criteria.json` saved, all quote checks pass.

**This notebook is mostly reading, not coding.** That's normal and it's the most valuable day in
the project.

---

## 02_cases.ipynb — invent 40 patients

Two functions, both defined in cells here:

```python
def oracle(spec):   # the right answers, from the rules in §3
def render(spec):   # turns a spec into readable sleep study / chart note / denial letter text
```

**Write `oracle()` from §3 before running any model, and then don't touch it.** If I adjust the
answer key after seeing what the model said, the answer key is worthless. This is the single
easiest way to accidentally cheat and it's very tempting later.

`oracle()` is just if-statements. `if spec["ahi"] is None: return "insufficient_evidence"` and so
on. Maybe 40 lines.

The 40 cases:

| Kind | How many | What's in them |
|---|---|---|
| Clearly qualifies | 8 | AHI 18–40, 30+ events, everything documented |
| Qualifies via the 5–14 route | 6 | AHI 7–13, 10+ events, one listed symptom |
| Clearly doesn't | 6 | AHI 3, no symptoms; E0471 with sleep apnea as the main diagnosis |
| **Not enough evidence** | 8 | AHI missing, event count missing, adherence never written down, symptom hinted at but never stated |
| Borderline | 8 | AHI exactly 14 with a symptom; AHI 15 but only 25 events; 4 hrs on 68% of nights; re-eval on day 95; E0470 with no record of E0601 failing |
| Nasty | 4 | denial letter cites the wrong rule; the record contains a sentence that *almost* matches the policy, to bait a fake quote |

Write 3–4 different phrasings per field so all 40 don't read identically.

**Then write 10 of the 40 by hand, in my own words, no template.** Score those separately later. If
the model does much worse on the handwritten ones, my templates were too easy and the README has
to say so.

**Sanity check on my answer key:** get a classmate to label 10 cases without seeing my answers,
then compute Cohen's kappa against `oracle()`. This checks whether my understanding of the policy
is sane, which is the right thing to be checking.

Save everything to `data/cases.json`.

**Done when:** 40 cases with answers, and I've read 5 of them out loud to check they don't all
sound the same.

---

## 03_pipeline.ipynb — build the thing, one cell at a time

This is where the actual pipeline gets built. Each piece is a function in a cell. Test each on 2–3
cases before moving to the next.

**Cell: chunking.** Two versions. Dumb one: cut everything into fixed 512-word blocks. Smart one:
one chunk per entry in `criteria.json`, and split the decoys on their headings. The smart one
exists so a two-part rule never gets sliced in half.

**Cell: embeddings + search.** Embed every chunk with `sentence-transformers`, stack into a numpy
array, and search with a dot product. ~200 chunks — no database needed, it's one matrix multiply.

**Cell: BM25.** Keyword search alongside the embeddings. Embeddings are fuzzy about exact strings
like `E0601`; BM25 isn't. Combine the two scores (normalize each to 0–1 first, then average —
they're on totally different scales otherwise).

**Cell: reranker.** Take the top ~20 from the combined search, re-score with a cross-encoder, keep
the best 5.

**Cell: ask the LLM.** Use a Pydantic model so the JSON comes back in a fixed shape:

```python
class Decision(BaseModel):
    criterion_id: str
    label: Literal["met", "unmet", "insufficient_evidence"]
    evidence_quote: str
    source_doc_id: str
    reasoning: str
```

Keep it flat. The one thing the prompt absolutely must say: **if the record is silent about
something, that's `insufficient_evidence`, not `unmet`.** Models get this wrong constantly
otherwise.

**Cell: check the quote.** Normalize both strings first — collapse whitespace, lowercase, convert
curly quotes `" "` to straight ones. Then:
- exact substring match → **verbatim**
- `rapidfuzz.partial_ratio >= 95` → **close enough**
- doesn't appear anywhere in any document → **made up**

Normalize *before* blaming the model. Curly quotes and line breaks cause most of the false
failures.

**Cell: abstain.** One line: if the quote didn't verify, the label becomes
`insufficient_evidence`.

**Done when:** I can run one case end to end and see a table of decisions with a verification
status on each.

---

## 04_experiments.ipynb — run it six ways

Copy the working functions from notebook 03 into the top of this notebook. Yes, copy-paste. It's
fine. What matters is that this notebook runs top to bottom in one go.

Configs are plain dicts in a cell:

```python
CONFIGS = [
    {"name": "row0_context_only", "retrieval": None,     "chunking": None,        "verify": True,  "abstain": False},
    {"name": "row1_naive",        "retrieval": "dense",  "chunking": "fixed",     "verify": False, "abstain": False},
    {"name": "row2_structure",    "retrieval": "dense",  "chunking": "criteria",  "verify": False, "abstain": False},
    {"name": "row3_hybrid",       "retrieval": "hybrid", "chunking": "criteria",  "verify": False, "abstain": False},
    {"name": "row4_rerank",       "retrieval": "rerank", "chunking": "criteria",  "verify": False, "abstain": False},
    {"name": "row5_full",         "retrieval": "rerank", "chunking": "criteria",  "verify": True,  "abstain": True},
]

for cfg in CONFIGS:
    results = run_all_cases(cfg)
    json.dump(results, open(f"data/results/{cfg['name']}.json", "w"), indent=2)
```

One `run_all_cases()` function, six dicts. Each row differs from the one above it by one setting —
that's what makes it an experiment instead of six random runs.

**Row 0 is required and I'm not allowed to skip it.** The whole policy fits in a modern context
window, so anyone looking at this will immediately ask "why bother with retrieval at all?" I run
it and publish the answer. **If it wins, I say so** — and note that it burns far more tokens per
case and doesn't scale past one policy, which is why the retrieval path exists. Volunteering a
baseline that beats me is the most credible thing I can put in this repo.

Then *Restart kernel → Run all* and let it finish before I trust anything.

**Done when:** six JSON files in `data/results/`.

---

## 05_results.ipynb — the numbers

Load the six result files and compute:

1. **Did it find the right rule?** (recall@5 — was the correct chunk in the top 5)
2. **Was it right?** macro-F1 across the three labels + a confusion matrix
3. **Were the quotes real?** verbatim rate, strict and fuzzy
4. **Did it invent clauses?** rate of quotes appearing nowhere in any document
5. **How often did it abstain, and was it more accurate on the ones it did answer?**

**Every number needs error bars.** With only 40 cases, `88%` is misleading — it sounds precise and
it isn't. Report `0.88 [0.74, 0.96]`. Use a Wilson interval for anything that's a proportion, and
a bootstrap (resample the 40 cases 1000 times) for F1.

Then the **risk–coverage curve**: sweep a confidence threshold, plot error rate on the questions it
answered against how many it answered. The shape of that curve is the whole argument for
abstention.

The table:

| # | What changed | Found right rule | F1 | Quotes real | Made up | Abstained | Accuracy when it answered |
|---|---|---|---|---|---|---|---|
| 0 | whole policy, no retrieval | — | | | | | |
| 1 | naive: fixed chunks, embeddings only | | | | | | |
| 2 | + chunk by rule | | | | | | |
| 3 | + BM25 | | | | | | |
| 4 | + reranker | | | | | | |
| 5 | + quote check + abstain | | | | | | |

**Done when:** every cell has a number with an interval.

---

## The letter, and a demo (last, and optional)

Feed the verified decisions into a letter template. Score it on **completeness, not writing
quality** — does it contain the HCPCS code, the AHI, the event count, the adherence numbers, the
requested duration, and the specific rule cited? Published work found LLM appeal letters read
nicely but leave out exactly these administrative details, so that's the interesting thing to
measure.

If I have time: a Gradio cell at the end of a notebook. Paste a denial letter and a record, see
the table and the draft. `demo.launch(share=True)` if I'm in Colab.

---

## When things go wrong

- **It's not finding the right rule (recall under 0.8)** → fix the chunking. Not the prompt, not
  the model. Chunking first, every time.
- **It says "unmet" when the record just doesn't mention something** → the prompt needs to spell
  out that absent information means insufficient evidence.
- **Quotes keep failing verification** → check my normalization before blaming the model. Curly
  quotes, non-breaking spaces, line breaks.
- **A number looks suspiciously good** → check the decoy documents are actually in the index.

## Using an LLM to help me build this

- Hand it **one function at a time**, with a description of the input and the output. "Here's what
  criteria.json looks like, write me a function that chunks the policy by it" — that works.
- Pasting this whole file and saying "build it" gets me something that runs and that I can't
  explain, which is the opposite of what I want.
- **Anything I can't explain out loud, I delete and rewrite.** Especially the Wilson interval and
  the normalization function — those are exactly the bits an interviewer pokes at.

## What I have to be honest about in the README

- The labels are **made up by me, not annotated by experts**. Constructed cases are easier than
  real ones, where charts are messy and contradictory in ways my generator doesn't reproduce.
- Synthetic patients only.
- **n = 40.** That's why everything has a confidence interval, and why the intervals are wide.
  Differences between rows are suggestive, not proof.
- One policy. Says nothing about how this behaves anywhere else.
- Disclaimer, verbatim:

> Not medical or legal advice. Not a clinical decision tool. Uses public Medicare policy and
> synthetic data only. Coverage rules change — always check the current LCD. Outputs may be wrong
> and must be reviewed by a qualified human.

## How I describe it in 20 seconds

> It helps a patient or clinician write an insurance appeal. It uses public Medicare rules and
> fake patients only. It cites the exact rule the patient meets, and if the record doesn't prove a
> rule, it says "not enough evidence" instead of making something up. I kept it to one policy —
> CPAP for sleep apnea — so I could actually measure whether it works.

## Out of scope, on purpose

A second policy. Real patient data. Filing anything. Commercial insurer policies. Fine-tuning
anything. Fancy eval frameworks. If everything above is finished and polished, the first thing
worth adding is entailment checking — not before.
