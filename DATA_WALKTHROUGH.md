# Follow the data — every stage, with the real bytes

`HOW_IT_WORKS.md` explains the ideas. This one shows the **actual data**, taken out of this repo at
each stage, with the arithmetic worked by hand.

Everything below is real except where a block is explicitly labelled *illustrative*. The numbers
come from `data/` and `data/results/full/`, and you can reproduce any of them.

We follow one rule (**B1**, the AHI threshold) and one patient (**case_024**) all the way through.

---

## Stage 0 — the raw input

A human opened the Medicare Coverage Database, saved the page, and dropped the file in `data/raw/`.
`L33718.html` is **296,360 characters** of page furniture wrapped around some policy text:

```html
<br><br>Hypopnea is defined as an abnormal respiratory event lasting at least 10
seconds associated with at least a 30% reduction in thoracoabdominal movement or
airflow as compared to baseline, and with at least a 4% decrease in oxygen
saturation.<br><br>The apnea-hypopnea index (AHI) is defined as the average number
of episodes of apnea and hypopnea per hour of sleep without the use of a positive
airway pressure device. For purposes of this policy, respiratory effort related
arousals (RERAs) a...
```

**What that paragraph is saying, medically.** An *apnea* is breathing stopping outright. A
*hypopnea* is breathing dropping without stopping — and Medicare pins it down with three numbers at
once: at least 10 seconds long, at least 30% less airflow, **and** at least a 4% drop in blood
oxygen. The *AHI* is those two event types added together and divided by hours of sleep. **RERAs**
are a milder event type, and the policy explicitly refuses to count them — which matters, because
counting them would inflate the index and make patients look sicker than the rule intends.

---

## Stage 1 — HTML to markdown

`01_policy.ipynb` strips the page furniture with `beautifulsoup` + `markdownify`. **296,360
characters become 42,465.** The same passage comes out as:

```
1. The beneficiary has an in-person clinical evaluation by the treating practitioner prior to the sleep test...
2. The beneficiary has a sleep test (as defined below) that meets either of the following criteria (1 or 2):

1. The apnea-hypopnea index (AHI) or Respiratory Disturbance Index (RDI) is greater than or equal to 15 events per hour with a minimum of 30 events; or,
2. The AHI or RDI is greater than or equal to 5 and less than or equal to 14 events per hour with a minimum of 10 events and documentation of:

1. Excessive daytime sleepiness, impaired cognition, mood disorders, or insomnia; or,
```

Thirteen documents go through this. No scraping — every page was saved by hand.

---

## Stage 2 — a human turns policy into a rule

This is the step that is **not** automated, and the most important one. Someone read L33718 and
wrote out 20 rules. Here is the entry for B1, exactly as stored:

```json
{
  "id": "B1",
  "summary": "AHI or RDI at least 15 per hour with at least 30 events",
  "text": "1. The apnea-hypopnea index (AHI) or Respiratory Disturbance Index (RDI) is greater than or equal to 15 events per hour with a minimum of 30 events; or,",
  "source": "L33718",
  "phase": "initial",
  "device": ["E0601", "E0470"],
  "spec_fields": ["ahi", "events"],
  "text_core": "The apnea-hypopnea index (AHI) or Respiratory Disturbance Index (RDI) is greater than or equal to 15 events per hour with a minimum of 30 events"
}
```

Field by field:

- **`text`** keeps the markdown list marker `1. ` and the trailing `; or,`. It is copy-pasted, so it
  is verifiable against the document character for character.
- **`text_core`** strips those. This is what gets **indexed and shown to the model**, because no
  model is going to reproduce a stray `; or,` and penalising it for that would measure nothing.
- **`device`** says this rule applies to CPAP and bi-level, not to `E0471`.
- **`spec_fields`** says the rule needs **two** patient facts: `ahi` *and* `events`. Remember that —
  it is the whole of Stage 3.

**Medically, why two numbers?** AHI is a *rate* — events per hour. A rate alone can mislead on a
short recording: three events in ten minutes is an AHI of 18, which sounds severe and proves
nothing. So the policy demands a rate **and** an absolute count. Both, or the rule is not met.

---

## Stage 3 — inventing a patient, and knowing the answer

`02_cases.ipynb` builds each patient from a **spec** — a dictionary of facts, where `null` means
*the record does not say*. Here is `case_024`:

```json
{
  "phase": "initial",
  "device": "E0601",
  "primary_dx": "OSA",
  "ahi": null,          ← the record never states an AHI
  "events": 142,
  "study_hours": null,
  "symptom": "insomnia",
  "swo_on_file": true
}
```

`render()` turns that into readable paperwork. Note what is **absent**:

```
SLEEP STUDY REPORT
A total of 142 respiratory events were recorded.
Study classification: Type I.
Ordered by the beneficiary's treating practitioner.
The study is a Medicare-valid sleep test.
The equipment used carries FDA approval for diagnostic sleep testing.
Testing was performed within the timeframe the policy requires.
Performed by a qualified sleep testing entity.
The facility satisfies state licensure requirements.
```

There is no AHI line. Not "AHI: not recorded" — **nothing**. Writing "not recorded" would hand the
model the answer; the absence makes it notice a gap on its own.

`oracle()` then computes the answer key from the spec:

```json
{
  "initial_evaluation": "met",
  "B1": "insufficient_evidence",
  "B2": "insufficient_evidence",
  "short_study_events": "insufficient_evidence",
  "sleep_test_valid": "met",
  "device_instruction": "met",
  "swo": "met"
}
```

**This is the trap the whole project is built around.** 142 events is far more than the 30 the rule
demands, so a careless reader — human or model — says *"142 > 30, met!"* But B1 needs the **rate**
too, and the study never gives one. You cannot compute events-per-hour without hours, and
`study_hours` is `null` as well.

So the honest answer is not "met" and not "unmet" — it is **"this record does not let me decide."**
Getting that distinction right is the thing being measured.

The code that produces it is deliberately boring:

```python
out["B1"] = _check(spec, ["ahi", "events"],
                   lambda: spec["ahi"] >= 15 and spec["events"] >= 30)
```

`_check` returns `insufficient_evidence` if **any** required field is `None`, before the numeric
test ever runs. Silence is checked first, always.

---

## Stage 4 — cutting documents into searchable pieces

Two strategies, and the comparison between them is the experiment.

**Fixed chunking** — 512 words, 64-word overlap, applied to all 13 documents:

```
207 chunks, mean 3,324 characters
```

**Criteria chunking** — one chunk per rule for L33718, headings for the other twelve:

```
143 chunks, mean 4,018 characters
  of which 20 are criterion chunks (one per rule)
```

Here is the failure mode that motivates the whole thing. A fixed chunk containing the AHI threshold,
taken from the index:

```
[NCD240.4::fix::1]
...with OSA if either of the following criterion using the AHI or RDI are met:
1. AHI or RDI greater than or equal to 15 events per hour, or 2. AHI or RD...
```

It is cut mid-rule — `2. AHI or RD` — and it is from the **wrong document**, the national
determination rather than the LCD being scored. Compare the criterion chunk:

```
[crit::B1]
The apnea-hypopnea index (AHI) or Respiratory Disturbance Index (RDI) is greater
than or equal to 15 events per hour with a minimum of 30 events
```

One clean sentence, labelled with the rule it belongs to. That label is what makes retrieval
measurable — `criterion_rank` looks for `crit::B1`, not for matching text, so a near-identical
sentence in a decoy cannot be mistaken for a hit.

---

## Stage 5 — embeddings, and what the multiplication actually is

An embedding model turns text into a fixed-length list of numbers, such that texts with similar
meaning end up pointing in similar directions.

Real shapes in this project, using `BAAI/bge-small-en-v1.5`:

```
each chunk  → a vector of 384 numbers
the index   → a matrix M of shape (143, 384)     # criteria chunking
a query     → a vector q of shape (384,)
scores      = M @ q   →  143 numbers, one per chunk
```

The vectors are **normalised to length 1**, which makes the dot product equal the cosine of the
angle between them: `1.0` = same direction, `0.0` = unrelated, `-1.0` = opposite.

### The arithmetic, in 4 dimensions instead of 384

*(Illustrative — the real vectors have 384 components and are not human-readable. The operation is
identical.)*

Say three chunks and one query embed as:

```
q      = [ 0.60,  0.70,  0.30,  0.25 ]     "AHI at least 15 with 30 events"

c_B1   = [ 0.58,  0.72,  0.28,  0.26 ]     the B1 rule
c_swo  = [ 0.10,  0.15,  0.90,  0.38 ]     the written-order rule
c_whl  = [-0.20,  0.05,  0.35,  0.91 ]     a wheelchair decoy
```

The score for each is the sum of element-wise products:

```
score(c_B1)  = 0.60·0.58 + 0.70·0.72 + 0.30·0.28 + 0.25·0.26
             = 0.348 + 0.504 + 0.084 + 0.065  =  1.001  → effectively 1.0, near-identical

score(c_swo) = 0.60·0.10 + 0.70·0.15 + 0.30·0.90 + 0.25·0.38
             = 0.060 + 0.105 + 0.270 + 0.095  =  0.530  → same topic area, different rule

score(c_whl) = 0.60·(-0.20) + 0.70·0.05 + 0.30·0.35 + 0.25·0.91
             = -0.120 + 0.035 + 0.105 + 0.2275 = 0.2475  → unrelated
```

Rank by score, take the top *k*. That is the entirety of "dense retrieval" — one matrix multiply and
a sort. With 143 chunks there is no database; it is `M @ q` on a numpy array.

**What the model is doing that a keyword search cannot:** `c_B1` scores high even though the query
never uses the words "apnea-hypopnea index" or "Respiratory Disturbance Index". It matches on
*meaning*. The flip side is Stage 6.

---

## Stage 6 — BM25, with real scores

BM25 is not AI. It is 1994-vintage keyword statistics, and it is in the pipeline because embeddings
are vague about exact strings — and codes like `E0601` are exactly where you want precision.

For a query term *t* in chunk *d*:

```
score contribution = idf(t) × ( tf · (k1+1) ) / ( tf + k1·(1 - b + b·|d|/avgdl) )

  tf    how many times t occurs in this chunk
  idf   how rare t is across the whole corpus — rare terms count for more
  |d|   this chunk's length
  avgdl average chunk length          k1 = 1.5, b = 0.75
```

The length term is the interesting one: dividing by `|d|/avgdl` is meant to stop long documents
winning just by being long.

### Real idf values from this index (N = 143 chunks, avg 586 tokens)

| term | appears in | idf | reading |
|---|---|---|---|
| `e0601` | 3 / 143 | **3.72** | rare, highly discriminating |
| `ahi` | 4 / 143 | **3.47** | rare, discriminating |
| `events` | 7 / 143 | 2.95 | fairly rare |
| `medicare` | 43 / 143 | 1.20 | common, weak signal |
| `the` | 95 / 143 | **0.41** | near-useless |

That table *is* the intuition: BM25 automatically learns that `e0601` is worth nine times more than
`the`, without anyone telling it, just from counting.

### And here is BM25 getting it wrong — for real

Query: `Medicare coverage rule for device E0601 during the initial phase. B1: AHI or RDI at least 15 per hour with at least 30 events`

```
1. NCD240.4::sec::2           score 54.10   "## Description Information  Benefit Category  Durab..."
2. crit::short_study_events   score 42.02   "If the AHI or RDI is calculated based on less than 2..."
3. crit::B2                   score 29.28   "The AHI or RDI is greater than or equal to 5 and les..."
4. crit::B1                   score 27.42   "The apnea-hypopnea index (AHI) or Respiratory Distu..."
5. A52467::sec::3             score 25.86   "### Article Guidance  Article Text  **NON-MEDICAL..."
```

**The rule we actually wanted is rank 4.** Rank 1 is a boilerplate "Description Information"
section — a long block that mentions *Medicare*, *coverage*, *device* and *initial* often enough to
accumulate score, while saying nothing about thresholds. The length penalty did not save us.

This is, concretely, why **row 3 (+BM25) has the worst F1 in the results table**. You can see the
mechanism in five lines of output.

---

## Stage 7 — combining the two, and reranking

Dense scores live around 0–1 (cosines). BM25 scores here run to 54. Averaging them raw would let
BM25 drown the embeddings entirely, so each is min-max normalised to 0–1 first:

```python
scores = w * _minmax(dense) + (1 - w) * _minmax(bm25_scores)   # w = 0.5
```

Then the **reranker**. Dense retrieval compares two vectors that were computed *separately* — the
query never saw the chunk. A cross-encoder reads them **together** as one input and scores the pair
directly. Far more accurate, far slower, so it only ever sees a shortlist:

```
24 candidates in  →  cross-encoder scores each (query, chunk) pair  →  keep top 3
```

Per rule, not per patient. With ~6 rules per case that is ~144 pair scorings per patient — which is
why the reranker configs are the slow ones, and why it is loaded lazily so rows 0–3 never pay for it.

**Result:** the reranker configs find the target rule in **232 of 239 decisions (97%)**, at a mean
rank of 1.04 — meaning when it finds the rule, it is almost always the *first* thing returned.

---

## Stage 8 — the prompt

The retrieved chunks, the rules to decide, and the patient's three documents are assembled into one
request. A real one from the cache:

```
prompt length: 79,256 characters
```

It opens like this:

```
POLICY TEXT -- the only place evidence_quote may come from.
Valid source_doc_id values: A52467, L33718, L33788, L33789, L33794, NCD240.4.1

[A52467] ### Article Guidance

Article Text

**NON-MEDICAL NECESSITY COVERAGE AND PAYMENT RULES**

For any item to be covered by Medicare, it must 1) be eligible for a defined
Medicare benefit category, 2) be reasonable and necessary for the diagnosis or
treatment of illness or injury...
```

Two things to notice. The valid `source_doc_id` values are **listed up front** — that line exists
because without it the model returned `source_doc_id: "CHART NOTE"`. And `L33788` (manual
wheelchairs) and `L33789` (power mobility) are in there: **decoys really do reach the prompt.**
Retrieval is not perfect, and the model has to ignore them.

---

## Stage 9 — what comes back

Schema-constrained JSON, so a missing field fails loudly:

```json
{
  "decisions": [
    {
      "criterion_id": "initial_evaluation",
      "label": "met",
      "evidence_quote": "For the initial in-person evaluation, the report would commonly document pertinent information about the following elements, but may include other details.",
      "source_doc_id": "A52467",
      "reasoning": "The chart note confirms an in-person clinical evaluation by the treating practitioner was completed before the sleep study.",
      "model_reported_confidence": 1
    },
    ...
  ]
}
```

Note `"model_reported_confidence": 1` — exactly 1.0. Across 239 decisions the mean is **0.995**, and
the gap between its confidence when right and when wrong is **+0.009**. It is recorded and never
trusted.

---

## Stage 10 — verification, character by character

Take the quote, take the document it claims to be from, normalise both, and look for one inside the
other. That is all:

```python
if normalize(quote) in normalize(source_text):
    return "supported"
```

### Why normalise at all

Two real examples from this corpus.

**The policy uses a symbol the model spells out.** L33718 contains:

```
Adherence to therapy is defined as use of PAP ≥4 hours per night on 70% of nights...
```

A model typing `>=4 hours` has quoted it correctly. Without normalisation that is a failed match and
a falsely accused model. So `≥ → >=` before comparing.

**Markdown wraps lines; models do not.** The policy text for `sleep_test_valid` is wrapped across
several lines in the file. The model returns it as one long line. Raw comparison fails; collapsing
all whitespace to single spaces succeeds. When I checked the 181 quotes that appear in drafted
letters, **139 matched raw and all 181 matched normalised** — every one of those 42 "failures" was
line wrapping.

`normalize()` does four things: NFKC unicode normalisation, curly quotes → straight, `≥`/`≤` →
`>=`/`<=`, all whitespace runs → one space, then lowercase.

### The five outcomes

| status | meaning |
|---|---|
| `supported` | exact substring of the claimed document |
| `close` | fuzzy match ≥ 95 but not exact |
| `wrong_doc` | real policy text, wrong document named |
| `from_record` | real text — but lifted from the **patient's own paperwork**, not a policy |
| `made_up` | in no policy and no record. Genuine fabrication. |

`from_record` exists because without it those decisions were counted as fabrication, reporting a
58% hallucination rate where the truth was about 4%.

---

## Stage 11 — turning 239 decisions into a score

Every decision is now a row with a `gold` label and a `predicted_final` label. Cross-tabulate them.

### The real confusion matrix, `row5_full`

```
gold \ predicted          met   unmet   insufficient
met                       172       1              0
unmet                       4      45              0
insufficient_evidence       5       1             11
```

Read it: of 173 truly-`met` decisions it got 172 right. Of 17 truly-`insufficient` it got 11, calling
5 of them `met` — **claiming evidence that was not there is the model's characteristic mistake**,
and it is the one that matters most in this domain.

### Precision, recall and F1 — computed by hand

For one class, using `unmet`:

```
tp = 45     it said unmet and it was unmet
fp =  2     it said unmet but it was not      (1 from met, 1 from insufficient)
fn =  4     it was unmet but it said something else

precision = tp/(tp+fp) = 45/47 = 0.9574     when it says unmet, how often is it right?
recall    = tp/(tp+fn) = 45/49 = 0.9184     of all real unmets, how many did it catch?
F1        = 2PR/(P+R)  = 2(0.9574)(0.9184)/(0.9574+0.9184) = 0.9375
```

All three classes:

| class | tp | fp | fn | precision | recall | F1 |
|---|---|---|---|---|---|---|
| met | 172 | 9 | 1 | 0.9503 | 0.9942 | **0.9718** |
| unmet | 45 | 2 | 4 | 0.9574 | 0.9184 | **0.9375** |
| insufficient_evidence | 11 | 0 | 6 | 1.0000 | 0.6471 | **0.7857** |

```
macro-F1 = (0.9718 + 0.9375 + 0.7857) / 3 = 0.8983
```

### Why macro-F1 and not accuracy

Plain accuracy is `228/239 = 0.9540`. That looks better — and it is the wrong number.

The gold labels are **173 met / 49 unmet / 17 insufficient**. A model that said "met" to everything
would score 72% accuracy while being useless. Macro-F1 averages the three classes **equally**, so
the 17 rare-but-critical `insufficient` cases carry the same weight as the 173 common ones. That is
why the headline is 0.8983 and not 0.9540: the `insufficient` class, with recall of only 0.647,
drags it down. It should.

---

## Stage 12 — error bars, worked by hand

Retrieval found the rule in **232 of 239** decisions. Point estimate `232/239 = 0.9707`. How
uncertain is that?

**The naive way** (normal approximation):

```
p ± 1.96·√(p(1-p)/n) = 0.9707 ± 1.96·√(0.9707 × 0.0293 / 239)
                     = 0.9707 ± 0.0214
                     = [0.9493, 0.9921]
```

Close to 1, this method misbehaves — push *p* a little higher and it produces an upper bound above
1.0, which is meaningless for a proportion.

**Wilson**, which is what the notebook uses:

```
              p + z²/2n           z              z²
centre =  ───────────────    half = ───── · √( p(1-p)/n + ──── )
              1 + z²/n           1+z²/n           4n²

→ [0.9408, 0.9857]
```

Asymmetric — more room below than above — which is correct behaviour near a boundary, and it can
never leave [0, 1].

**For F1 there is no closed form**, so it is bootstrapped: resample and recompute 1,000 times, then
take the 2.5th and 97.5th percentiles. Critically, it resamples the **40 patients**, not the 239
decisions — all ~6 decisions for one patient come from the same retrieved context and the same single
model call, so they are correlated. Treating them as independent would report intervals tighter than
the evidence supports.

---

## The whole path, for one rule and one patient

```
L33718.html                    296,360 chars of HTML
      │ beautifulsoup + markdownify
      ▼
L33718.md                       42,465 chars of markdown
      │ a human reads it and writes down 20 rules
      ▼
criteria.json → "B1"            text_core + phase + device + spec_fields
      │ chunk_by_criteria
      ▼
crit::B1                        one chunk, one sentence, labelled
      │ embed → 384 numbers, one row of a (143, 384) matrix
      ▼
search(query) = M @ q           143 scores, sorted
      │ + BM25, min-max normalised, averaged
      │ + cross-encoder reranks 24 → 3
      ▼
prompt                          79,256 chars: policy + rules + patient paperwork
      │ one API call per patient
      ▼
{"criterion_id": "B1", "label": ..., "evidence_quote": ...}
      │ normalize() both sides, substring test
      ▼
quote_status = supported        ← not AI. A string match.
      │ unverified → insufficient_evidence
      ▼
row: gold vs predicted_final
      │ × 239 decisions × 6 configs
      ▼
confusion matrix → P, R, F1 per class → macro-F1 = 0.8983
      │ Wilson for proportions, cluster bootstrap for F1
      ▼
0.90 [0.82, 0.95]
```

---

## Reproduce any of it

```python
import json
crit  = {c["id"]: c for c in json.load(open("data/criteria.json"))}
cases = {c["id"]: c for c in json.load(open("data/cases.json"))}
rows  = json.load(open("data/results/full/row5_full.json"))["rows"]

crit["B1"]["text_core"]                                   # the rule
cases["case_024"]["spec"]                                 # the patient
cases["case_024"]["documents"]["sleep_study"]             # what the model saw
cases["case_024"]["gold"]                                 # the answer key

[r for r in rows if r["case_id"] == "case_024"]           # what it decided, and why
```

Each row carries `evidence_quote`, `quote_status`, `retrieval_rank`, `reasoning` and
`citation_scope` — the full audit trail for a single decision. Start there when a number surprises
you: in this project, three times out of three, a surprising number turned out to be a broken
measurement rather than a broken model.
