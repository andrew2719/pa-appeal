# How it works — the whole project in plain terms

A companion to the notebooks. This explains *what* happens and *why*; the notebooks show *how*.

If a term is unfamiliar, jump to **§10** — medical, Medicare and machine-learning terms are
each explained separately, with a note on where they appear in this project. **§9** covers how
a real Medicare appeal works and how little of it this models.

**Is notebook 06 enough on its own?** Not quite. `06_walkthrough.ipynb` is the code path with
commentary — it is excellent once you already know what you are looking at. Read this document
first, then run 06 with the code in front of you. This file is the map; 06 is the territory.

---

## 1. The problem, in one paragraph

Medicare publishes a public rulebook saying when it will pay for a CPAP machine. When a claim is
denied, someone has to argue back — and the argument has to point at specific rules. The project
takes a (fake) patient's paperwork and a (fake) denial letter, walks the rulebook one rule at a
time, and decides for each rule: **met**, **unmet**, or **not enough evidence**. Every met/unmet
must come with a quote from the policy, and that quote is checked by literally searching for it in
the policy text.

**The letter was never the point.** The point was measuring four things:

1. How often does it find the right rule?
2. Are the quotes real?
3. Does it invent rules that do not exist?
4. Does accuracy improve when it is allowed to say "I don't know"?

---

## 2. The inputs

Three kinds of data, and **only one of them is real**.

### 2.1 The rulebook — real, public, downloaded by hand

Thirteen Medicare documents, saved as HTML and converted to markdown. They split three ways, and
this distinction matters when reading the results:

| group | documents | role |
|---|---|---|
| **target** | `L33718` (PAP devices for sleep apnea) | the policy the whole project is about |
| **supporting** | `A52467`, `A55426`, `NCD 240.4`, `NCD 240.4.1` | the coding article, documentation rules, and two national determinations |
| **decoy** | 8 unrelated equipment LCDs | oxygen, hospital beds, wheelchairs, nebulizers, glucose monitors, surgical dressings, infusion pumps, power mobility |

**Why the decoys exist.** If the search index contains only one document, then "did it find the
right document?" is always *yes*, and that measurement is worthless. The decoys make retrieval a
real question — the system has to pick CPAP rules out of a pile that also contains wheelchair rules.

### 2.2 The rules — a human reading with a pen

`data/criteria.json` holds **20 rules** extracted from L33718 by hand. Each one carries:

- `text` — the sentence, **copy-pasted character for character** from the policy
- `text_core` — the same sentence without the markdown list marker (this is what gets indexed)
- `phase` — `definition` / `initial` / `continued` / `general`
- `device` — which HCPCS codes it applies to (`E0601`, `E0470`, `E0471`), or nothing = all
- `spec_fields` — which patient facts the rule needs

This file does **two jobs at once**, which is why it is the most important file in the repo:

1. it is how the policy gets cut into searchable chunks, and
2. it is the answer key that retrieval is scored against.

If a quote were even slightly wrong — a retyped word, a dropped comma — every number downstream
would be quietly wrong and nothing would flag it. So the notebooks assert all 20 are verbatim
before doing anything else.

**Five of the 20 are definitions** (what "apnea" means, what "AHI" means). A patient record cannot
*satisfy* a definition, so they are never given a label. They stay in the file because retrieval
benefits from having them — when deciding an AHI threshold, the definition of AHI is useful
context. That leaves **15 scoreable rules**.

### 2.3 The patients — completely invented

**40 synthetic patients.** No real patient data at any point.

A Python function invents each one, so **the correct answer is known by construction** — no
clinician labelling, no annotation. They fall into six buckets:

| bucket | n | what is in them |
|---|---|---|
| clearly qualifies | 8 | AHI 18–40, 30+ events, everything documented |
| qualifies via the 5–14 route | 6 | AHI 7–13, 10+ events, one listed symptom |
| clearly does not | 6 | AHI 3, no symptoms; E0471 billed against an OSA diagnosis |
| **not enough evidence** | 8 | AHI missing, events missing, adherence never written down |
| borderline | 8 | AHI exactly 14; AHI 15 but only 25 events; 68% of nights; re-eval on day 95 |
| adversarial | 4 | denial letter cites the wrong rule; near-miss sentences planted to bait a fake quote |

Each patient produces three documents: a **sleep study**, a **chart note**, and a **denial letter**.

**The single most important convention in the project:** a field set to `None` means *the record
does not say*, and the generator then **omits it entirely**. It does not write "AHI: not
documented" — it writes nothing, because saying "not documented" would hand the model the answer
instead of making it notice a gap.

That is the distinction the whole project measures: **silence versus contradiction.** "AHI is 3"
means the patient fails. "No AHI anywhere" means you cannot tell. Conflating those two is the
failure mode this is built to detect.

---

## 3. Where the AI actually is

You asked specifically, and it is worth being precise, because most of this system is not AI.

| piece | AI? | what it actually is | trained by us? |
|---|---|---|---|
| HTML → markdown | no | rule-based parsing (`beautifulsoup`, `markdownify`) | — |
| `criteria.json` | no | a person reading a policy with a pen | — |
| case generator | no | Python if-statements and phrase templates | — |
| **`oracle()`** — the answer key | no | ~60 lines of if-statements | — |
| chunking | no | string splitting | — |
| **embeddings** | **yes** | neural encoder, `BAAI/bge-small-en-v1.5` (~33M params) | no — downloaded |
| BM25 | no | classical keyword statistics, from 1994 | — |
| **reranker** | **yes** | neural cross-encoder, `BAAI/bge-reranker-base` (~278M params) | no — downloaded |
| **the decision** | **yes** | an LLM, `gemini-3.1-flash-lite`, via API | no |
| **quote verification** | **no** | `if quote in policy_text` | — |
| abstention | no | one if-statement | — |
| metrics | no | arithmetic and two confidence intervals | — |

**Nothing in this project was trained.** There is no training loop, no fine-tuning, no GPU
requirement. Three neural components, all pre-trained downloads, one of them behind an API.

Two things are deliberately *not* AI, and that is the point:

- **The answer key is if-statements.** If it were a model, you would be measuring one model
  against another and could never say which was wrong.
- **Quote verification is string matching.** If a second LLM checked the quotes, "are the quotes
  real?" would inherit that model's errors. Because it is `in`, the number is not arguable.

---

## 4. The pipeline, stage by stage

For one patient:

```
patient documents ─┐
                   ├─→ retrieve rules → prompt → LLM → JSON decisions
policy chunks ─────┘                                        │
                                                            ▼
                                              verify each quote (string match)
                                                            │
                                                            ▼
                                            unverified? → "not enough evidence"
                                                            │
                                                            ▼
                                                    compare to answer key
```

### 4.1 Which rules even apply

Before anything else: `applies_to()` decides whether a rule is in play, using `phase` and `device`
from `criteria.json`.

An E0470-only rule is never put to a CPAP patient. A continued-coverage rule is never put to a
patient in their first month. **A rule that does not apply has no honest met/unmet answer**, and
scoring one measures nothing.

This also means criterion *selection* is **given, not predicted** — the pipeline is told which
rules apply. Gold *labels* never enter the prompt. So what is measured is **adjudication** (can it
judge a rule correctly), not **triage** (can it work out which rules matter). That is a real
limitation and it is stated in the README.

### 4.2 Chunking — cutting the policy into searchable pieces

Two strategies, because comparing them *is* the experiment:

- **fixed** — every document cut into 512-word blocks with 64-word overlap → **207 chunks**. The
  dumb baseline. A rule can get sliced in half.
- **criteria** — one chunk per rule for L33718, headings for the other twelve documents → **143
  chunks**. A two-part rule stays whole.

### 4.3 Retrieval — finding the right rule

Three techniques, each building on the last:

1. **Dense embeddings.** Every chunk is turned into a vector of numbers by a neural encoder such
   that similar meanings land near each other. The query gets the same treatment, and similarity
   is a dot product. With ~200 chunks the "vector database" is one numpy array.
2. **BM25.** Classical keyword scoring. It is in there because embeddings are *fuzzy about exact
   strings* — codes like `E0601` are precisely where a keyword match beats a semantic one. The two
   scores are normalised to 0–1 before averaging, because they are on completely different scales.
3. **Cross-encoder reranker.** Takes the top 24 candidates and re-scores them by reading the query
   and the passage *together* rather than embedding them separately. Slower, much sharper. Keeps
   the best 3.

**The single most important design decision in the project:** retrieval runs **once per rule**,
not once per patient. Each rule gets its own short query, and the results are merged into one
context. The first version used one blended query naming all six rules at once — and **18 of 18
criterion passages were never retrieved**, because twenty one-sentence rules cannot outrank 123
long decoy sections on a query about six things simultaneously.

Fixing that took target-rule retrieval from **0 to 25/26**. Crucially it costs **no extra API
calls** — retrieval is local and free; only the single model call per patient costs money.

### 4.4 The model call — one per patient

The retrieved policy text, the list of rules to decide, and the patient's three documents go into
one prompt. A schema forces the reply into a fixed shape, so a missing field fails loudly instead
of silently becoming `None` three steps later. For each rule the model returns:

- `label` — met / unmet / insufficient_evidence
- `evidence_quote` — a sentence copied from the policy text supplied
- `source_doc_id` — which document it came from
- `reasoning` — at most two sentences
- `model_reported_confidence` — 0–1 (recorded, never trusted; see §6.5)

Every response is **cached on disk by prompt hash**, so re-running costs nothing. The flip side:
changing one character of the system prompt invalidates the whole cache.

### 4.5 Verification — the part that is not AI

Both strings are normalised first (collapse whitespace, straighten curly quotes, `≥` → `>=`,
lowercase), because curly quotes and line wrapping cause most of the quotes that *look* wrong and
are not. Then:

| status | meaning |
|---|---|
| `supported` | exact substring of the claimed document |
| `close` | fuzzy match ≥ 95 |
| `wrong_doc` | real policy text, but a different document than claimed |
| `from_record` | real text — copied from the **patient record**, not a policy |
| `made_up` | appears in no policy and no record. Actual fabrication. |
| `empty` | no quote offered |

### 4.6 Abstention — one if-statement

If a met/unmet decision's quote did not verify, the label becomes `insufficient_evidence`. That is
the whole mechanism. It cannot be argued with, because it is not a judgement.

---

## 5. How it was evaluated

This is the part that makes it an experiment rather than a demo.

### 5.1 The answer key is frozen

`oracle()` was written from the policy **before any model output was seen**, and never edited
afterwards. Adjusting an answer key after seeing results is the easiest way to accidentally cheat,
and it is very tempting once a disagreement shows up.

It did show up — see §7.3 — and the oracle still was not touched. Instead the disagreement is
reported both ways, with the frozen number as the headline.

### 5.2 The ablation — six configurations, one change at a time

| # | config | what changed |
|---|---|---|
| 0 | context only | whole policy in the prompt, no retrieval |
| 1 | naive | fixed 512-word chunks, embeddings only |
| 2 | structure | + chunk by rule |
| 3 | hybrid | + BM25 |
| 4 | rerank | + cross-encoder reranker |
| 5 | full | + quote verification and abstention |

Each row differs from the one above by **exactly one setting**. That is what lets you attribute a
change to a cause. Six random configurations would tell you nothing.

**Row 0 is required and cannot be skipped.** L33718 fits in a modern context window, so the
obvious question is "why build retrieval at all?" Publishing a baseline that might beat you is the
most credible thing in the repo.

Each configuration runs over the same **40 patients = 239 decisions**, so **1,434 decisions total**.

### 5.3 What each metric measures

| metric | question it answers | how |
|---|---|---|
| **found right rule** | did retrieval surface the rule's own passage? | rank of the criterion's chunk within its own results |
| **F1 (macro)** | is the label correct, across all three labels equally? | scikit-learn, averaged over met/unmet/insufficient |
| **quotes real** | is the citation genuinely in the document claimed? | string match |
| **made up** | did it invent a clause? | not in any policy *and* not in the record |
| **abstained** | how often did it answer "insufficient_evidence"? | count |
| **acc. when answered** | accuracy on decisions where it committed | excludes insufficient predictions |

**Why macro-F1 rather than accuracy.** The gold labels are imbalanced — 173 met, 49 unmet, 17
insufficient. Plain accuracy would let a model score well by always saying "met". Macro-F1 averages
the three classes equally, so the rare-but-important `insufficient_evidence` class carries the same
weight as the common one.

A caution that bit us: **macro-F1 over three labels is capped at (classes present)/3.** Inside a
bucket containing only two of the labels it cannot exceed 0.67 — which looks like a bad score and
is not one. Per-bucket reporting now shows accuracy alongside, with the cap flagged.

### 5.4 Error bars, and why there are two kinds

With 40 cases, "88%" is misleading — it sounds precise and is not.

- **Wilson intervals** for proportions. The textbook normal approximation misbehaves at small n and
  near 0 or 1 (it can produce intervals below zero); Wilson does not.
- **A cluster bootstrap over cases** for F1. Resample the **40 patients**, not the 239 decisions.
  Every decision for one patient comes from the same retrieved context and the same single model
  call, so decisions within a patient are **correlated**. Treating 239 correlated observations as
  239 independent ones reports intervals far tighter than the evidence supports — the classic way
  to make a small study look precise.

### 5.5 A separate question: does the citation point at the *right* rule?

"Quotes real" establishes **provenance** — the sentence is genuinely in the document named. That is
all a string match can establish. **It does not show the sentence supports the label**, and nothing
in the project checks that; no amount of string matching could.

So a second diagnostic classifies each verified citation:

- **own_rule** — inside that rule's canonical sentence
- **other_rule** — matches a *different* rule's sentence
- **policy_prose** — real policy text, but not one of the 20 curated sentences

In the final configuration: **225/231 = 97.4%** of citations quote the exact rule being decided.

It is reported as a diagnostic and **deliberately does not trigger abstention** — see §6.4 for why.

---

## 6. What the results actually say

| # | config | found rule | F1 | quotes real | made up |
|---|---|---|---|---|---|
| 0 | whole policy | — | 0.86 [0.77, 0.92] | 0.87 | 0.05 |
| 1 | naive | 0.34 [0.28, 0.40] | 0.81 [0.72, 0.89] | 0.89 | 0.02 |
| 2 | + chunk by rule | 0.65 [0.59, 0.71] | 0.91 [0.82, 0.97] | 0.95 | 0.02 |
| 3 | + BM25 | 0.67 [0.61, 0.73] | 0.79 [0.72, 0.85] | 0.82 | 0.00 |
| 4 | + reranker | **0.97 [0.94, 0.99]** | 0.90 [0.82, 0.95] | **0.97** | 0.00 |
| 5 | + verify + abstain | 0.97 | 0.90 | 0.97 | 0.00 |

### 6.1 What is solid

**Retrieval improves monotonically: 0.34 → 0.65 → 0.67 → 0.97, with non-overlapping intervals.**
Chunking by rule is worth 31 points over fixed chunks; the reranker another 30 on top. This is the
one result to state without hedging.

### 6.2 What is not

**The F1 column cannot rank the configurations.** Every interval overlaps every other one, and row
0 — no retrieval at all — sits inside all of them. With 40 cases that column is not precise enough
to order anything. Saying so is more credible than picking a winner.

### 6.3 BM25 hurts

Row 3 has the worst F1 (0.79) and worst quote rate (0.82) despite retrieving slightly better than
row 2. Lexical matching surfaces passages that share vocabulary with the query but state a
different rule. An unexplained negative result, and the most interesting thing left to chase.

### 6.4 Abstention never fires — and that reverses the original thesis

In row 5, **zero** decisions out of 239 were changed by the quote check. Once retrieval and the
prompt were fixed there was nothing left to catch, so row 5 is identical to row 4 on everything.

Worse for the original hypothesis: across all six configs, **23 decisions cited a quote found in no
policy and no record — and all 23 had the correct label.** Decisions where the model honestly
returned an empty quote were only ~69% correct.

So: **quote verification guarantees every citation shown to a human is real. It does not identify
wrong answers.** On this data the unverifiable answers were disproportionately the *right* ones, and
abstaining on them would have discarded 23 correct decisions to remove zero incorrect ones.

That is a negative result the project did not set out to find, which makes it worth more than the
positive one it expected.

### 6.5 The model's confidence is worthless, and that is measured

Mean self-reported confidence **0.995**. When correct: 0.995. When wrong: 0.986. **Gap: +0.009.**
The model is as confident when wrong as when right. It is recorded as an exploratory field and
never used to rank, threshold, or abstain.

### 6.6 Where the errors are

Per-criterion accuracy is near ceiling — `swo` 40/40, `device_instruction` 31/31,
`initial_evaluation` 31/31. By bucket: adversarial **1.00**, clear_unmet **1.00**, met_5_14
**1.00**, clear_met 0.96, borderline 0.94, **insufficient 0.87**.

The hardest bucket is the one where the record is silent — which is exactly the distinction the
project exists to test, so it is the right thing to be hardest.

The weakest rule is `short_study_events` (1/3): on a study under two hours the event minimum drops
from 30 to 10 when a symptom is documented, and the model applied the higher threshold and missed
the exception.

### 6.7 The letter

Built from **verified decisions only** — a rule reaches the letter only if labelled `met` *and* its
citation verified. Across 40 cases: **181 policy sentences, every one verbatim, zero unverified
citations reaching a human.**

Scored on **completeness, not writing quality**:

| drafter | facts present | complete letters |
|---|---|---|
| template | 210/210 = 100% | 40/40 |
| model | 204/210 = 97% | 34/40 |

The template's 100% is the floor, not a result. The model's letters read better and drop exactly
one thing: the **verbatim policy quotation, missing from 6 of 40** — the administrative detail a
claims reviewer actually needs.

---

## 7. What actually made it work

Four changes moved the numbers. Three were bugs; one was a prompt.

### 7.1 Per-criterion retrieval (the big one)

One query per rule instead of one blended query per patient. Target-rule coverage **0/18 → 25/26**.
Everything downstream that looked like a model failure was really this, reported four ways.

### 7.2 Fencing the record off from the policy

The first prompt headed the record sections `SLEEP STUDY:` and `CHART NOTE:` — which read like
document names. The model quoted them and returned `source_doc_id="CHART NOTE"`. The verifier only
knows policies, so those came back `made_up`, and the reported fabrication rate was **58% when the
true rate was about 4%**.

Fixing it meant listing the valid document ids up front and labelling the record explicitly as
not-policy. Record-quoting went to **zero**.

### 7.3 Telling the model how to score a rule it does not qualify under

The model kept returning `insufficient_evidence` with the reasoning *"this criterion is not
applicable because the patient met B1"*. B1 and B2 really are alternative routes, so it was not
being stupid — there is simply no not-applicable label.

The oracle scores each rule **as written**, so a patient with AHI 26 has B1 met and B2 unmet. The
convention was documented in notebook 02 but never communicated to the model. Adding one rule —
*there is no "not applicable" option; `insufficient_evidence` means the record does not say* —
fixed it without leaking any threshold. `unmet` went from appearing once in 210 decisions to 21 of
24 alternative-route cases correct.

This is a fair fix, not cheating: it specifies the task, it does not supply answers.

### 7.4 The habit that caught all three

Each of those was found the same way. The tell was never that the number was bad — it was that the
number was **suspiciously uniform**:

| it looked like | it actually was |
|---|---|
| fabricates 58% of citations | quoted the patient record; verifier could not see the record |
| retrieval collapses, F1 → 0.07 | one blended query per case instead of one per rule |
| omits the request from **every** letter | the checklist demanded one particular synonym |

58% concentrated in one status. 18 of 18 passages unretrieved. 40 of 40 letters failing one item.
**Real model failures are ragged. Clean, total failure usually means the instrument is wrong.**

> When a result is both surprising and uniform, check the instrument before you believe it.

That is the most transferable thing in the project.

---

## 8. What it cannot do

Be able to say these without being prompted:

- **Labels are mine, not a clinician's.** Constructed cases are cleaner than real charts, which
  contradict themselves in ways the generator does not reproduce.
- **Every case is templated.** Each field is phrased from a bank of 3–4 variants, so the model
  could be matching phrasing rather than reading. The planned control — ten cases rewritten by hand
  — was never done, so the accuracy figures are an **upper bound**.
- **n = 40**, one policy, one model, one revision of that policy.
- **Criterion selection is given, not predicted.** This measures adjudication, not triage.
- **String verification proves provenance, not logical support.** A quote can be genuinely present
  and still not justify the label. Nothing here checks entailment.
- No claim that this is clinically valid, legally sufficient, or that it reduces denials.

---

## 9. The real world — what actually happens, and what this models

Worth understanding, because it tells you how small a slice of the real process this is.

### 9.1 A note on the repo name

The repo is `pa-appeal`, where PA suggests **prior authorization** — asking the payer to approve
something *before* it is supplied. That is not quite what this models.

What the generated denial letters actually describe is a **post-service claim denial**: the
equipment was supplied, the supplier billed Medicare, and Medicare refused to pay. The response to
that is a **redetermination** request — the first rung of the appeals ladder. Some DMEPOS items do
require prior authorization, but the mechanics modelled here are the appeal of a denied claim.

### 9.2 Who the parties are

| party | role |
|---|---|
| **beneficiary** | the patient. Medicare's term for the person covered. |
| **treating practitioner** | the clinician managing the patient. The policy repeatedly requires *this* person to have done the evaluation, ordered the test, and reviewed the data — not just any physician. |
| **DME supplier** | the company that actually provides the CPAP machine and bills Medicare. Usually the party that gets denied and appeals. |
| **DME MAC** | Durable Medical Equipment Medicare Administrative Contractor. A private company contracted by CMS to process claims for a region, and the author of the LCD. |
| **CMS** | Centers for Medicare & Medicaid Services. The federal agency. Issues NCDs; contracts the MACs. |

### 9.3 What a real denial and appeal look like

1. **The claim is submitted** by the supplier, with HCPCS codes and diagnosis codes.
2. **It is denied.** The notice carries reason codes and boilerplate, and typically asserts a
   conclusion — *"documentation does not support medical necessity"* — without saying which sentence
   of which policy was not satisfied. That vagueness is precisely the gap this project attacks.
3. **Someone assembles the record**: the sleep study, the chart notes proving the face-to-face
   evaluation, the order, the adherence download, proof the supplier gave instruction.
4. **Someone argues it against the policy**, rule by rule. This is the slow, expert, unglamorous
   part — and it is the part the project automates.
5. **It is filed**, and the clock starts.

### 9.4 The five levels of Medicare appeal

| level | called | decided by |
|---|---|---|
| 1 | **Redetermination** | the same MAC that denied it, different reviewer |
| 2 | **Reconsideration** | a QIC — Qualified Independent Contractor |
| 3 | **ALJ hearing** | an Administrative Law Judge at OMHA |
| 4 | **Council review** | the Medicare Appeals Council |
| 5 | **Judicial review** | federal district court |

Each level has a filing deadline and a decision deadline, and levels 3 and 5 have a minimum
amount-in-controversy that adjusts annually. **Verify current figures against CMS guidance** —
commonly cited is 120 days to request a redetermination and 60 days for the MAC to decide, but
these are exactly the kind of number that changes, and the project's own rule applies: check the
current source.

**This project models level 1 only**, and only the reasoning inside it.

### 9.5 What is modelled versus skipped

| modelled | skipped entirely |
|---|---|
| reading the policy rule by rule | submitting anything, to anyone |
| deciding met / unmet / not enough evidence | deadlines, filing windows, amount in controversy |
| citing the specific sentence relied on | claim forms, reason codes, modifiers |
| naming what the record does *not* establish | MAC jurisdiction and regional variation |
| drafting a letter for a human to review | levels 2 through 5 |
| | real patients, real claims, real money |

### 9.6 Why this slice is the interesting one

The bottleneck in a real appeal is not writing prose — a template does that. It is **going through a
40-page policy, deciding which of its rules this patient satisfies, and proving each one with the
exact sentence**. That is slow, requires expertise, and is where mistakes are expensive.

It is also exactly where a language model is most tempting and most dangerous: it will produce a
confident, fluent, plausible answer whether or not the policy says what it claims. Hence the
project's actual shape — the model proposes, and a string match disposes.

---

## 10. Every term, explained

### 10.1 Medical and clinical

| term | what it means | where it shows up here |
|---|---|---|
| **OSA** | Obstructive Sleep Apnea. Breathing repeatedly stops during sleep because the airway collapses. | the condition the whole policy is about |
| **apnea** | breathing stops for **≥ 10 seconds** | `apnea_def` criterion |
| **hypopnea** | breathing does not stop but drops — ≥ 10 s, ≥ 30% reduction in airflow, with ≥ 4% oxygen desaturation | `hypopnea_def` |
| **AHI** | Apnea-Hypopnea Index — apneas + hypopneas **per hour of sleep**. The severity number. | the threshold B1 and B2 turn on |
| **RDI** | Respiratory Disturbance Index — similar, but per hour of *recording* rather than sleep. Home tests report this because they cannot measure true sleep time. | treated as interchangeable with AHI by the policy |
| **RERA** | Respiratory Effort Related Arousal — a lesser event. The policy **excludes** these from AHI. | `ahi_def` says so explicitly |
| **polysomnogram** | full overnight sleep study in a lab, wired up | a Type I study |
| **Type I–IV study** | sleep-test tiers. Type I = attended lab study; II–IV = progressively simpler home tests. | `sleep_test_valid`; only I and II measure true sleep time |
| **CPAP** | Continuous Positive Airway Pressure — one steady pressure holding the airway open | HCPCS `E0601` |
| **bi-level** | two pressures, higher on inhale, lower on exhale. Easier to breathe against. | `E0470` |
| **backup rate** | the device forces a breath if the patient does not take one. For patients who stop *trying* to breathe, not just those whose airway collapses. | `E0471` — **not covered when the primary diagnosis is OSA**, because OSA is an obstruction problem, not a drive problem |
| **titration** | tuning the pressure to the individual patient | part of what "optimal therapy" means in `E0470_ineffective` |
| **adherence / compliance** | actually using the machine. Medicare defines it: **≥ 4 hours per night on 70% of nights over 30 consecutive days.** | the `adherence` criterion |
| **EDS** | Excessive Daytime Sleepiness | one of the qualifying symptoms in the B2 route |

### 10.2 Medicare, product and administrative

| term | what it means | where it shows up here |
|---|---|---|
| **LCD** | Local Coverage Determination — a MAC's regional rule on when something is reasonable and necessary | `L33718` is the one this project is about |
| **NCD** | National Coverage Determination — a nationwide CMS rule. Sits above LCDs. | `NCD 240.4`, `NCD 240.4.1` |
| **Policy Article** | companion to an LCD carrying coding, billing and documentation detail | `A52467`, `A55426` |
| **DMEPOS** | Durable Medical Equipment, Prosthetics, Orthotics and Supplies — the category CPAP falls in | why the decoys are all equipment policies |
| **HCPCS** | the equipment billing code set. Free to use. | `E0601`, `E0470`, `E0471` |
| **CPT** | the *procedure* code set — **AMA-licensed and copyrighted**. Deliberately absent from this repo. | a stated project rule |
| **SWO** | Standard Written Order. Must reach the supplier **before** the claim is submitted, or the claim is denied. | the `swo` criterion — scored on all 40 cases |
| **date of service** | when the item was supplied. Determines *which revision* of the policy applies. | why the README records L33718 revision R11 |
| **reasonable and necessary** | the statutory test for Medicare coverage. LCDs exist to say what it means in practice. | the phrase the denial letters use |
| **medical necessity** | shorthand for the same idea | what a denial typically claims is missing |
| **redetermination** | the level-1 appeal | what the drafted letters request |
| **initial vs continued coverage** | first 3 months versus beyond. Continuing requires a re-evaluation between day 31 and 91 *and* documented adherence. | the `phase` field on every case |

### 10.3 Machine learning and retrieval

| term | what it means | where it shows up here |
|---|---|---|
| **LLM** | Large Language Model. Predicts text. Fluent, confident, and not a database. | `gemini-3.1-flash-lite` makes the decisions |
| **token** | roughly a word-piece. Models are priced and limited in tokens. | why row 0 "burns more tokens per case" |
| **context window** | how much text fits in one request | L33718 fits, which is why row 0 is possible at all |
| **prompt** | what you send | built fresh per patient |
| **system prompt** | standing instructions, separate from the per-request content | the numbered rules — where two bug fixes live |
| **temperature** | randomness. 0 = most deterministic. | set to 0, so re-runs are reproducible |
| **structured output / schema** | forcing the reply into a fixed JSON shape | so a missing field errors loudly instead of silently |
| **hallucination** | fluent, confident, false output | the thing quote verification exists to catch |
| **grounding** | tying output to supplied source text | the retrieved policy chunks |
| **RAG** | Retrieval-Augmented Generation — fetch relevant text, put it in the prompt, answer from it | rows 1–5 |
| **chunk** | one searchable slice of a document | 207 fixed-size, or 143 rule-shaped |
| **embedding** | a list of numbers representing meaning, such that similar meanings sit close together | `bge-small-en-v1.5` produces them |
| **dense retrieval** | search by embedding similarity — good at meaning, fuzzy about exact strings | rows 2 and up |
| **sparse retrieval / BM25** | search by keyword statistics — good at exact strings like `E0601` | row 3 adds it, and it **hurt** |
| **hybrid search** | combine both, after normalising the scores to a common scale | `search(mode="hybrid")` |
| **cross-encoder / reranker** | reads query and passage *together* and scores the pair. Slower, sharper. | `bge-reranker-base`; the single biggest retrieval gain |
| **recall@k** | was the right thing in the top *k* results? | the "found right rule" column |
| **pre-trained** | shipped already trained; you download and use it | all three neural parts here. **Nothing was trained in this project.** |

### 10.4 Evaluation and statistics

| term | what it means | where it shows up here |
|---|---|---|
| **ground truth / gold label** | the known-correct answer | produced by `oracle()` |
| **oracle** | code that computes the right answer because you built the data | 60 lines of if-statements, frozen before seeing output |
| **class** | one possible label | `met`, `unmet`, `insufficient_evidence` |
| **class imbalance** | classes appearing at very different rates | 173 / 49 / 17 — which is why plain accuracy would mislead |
| **accuracy** | fraction correct. Easy to read, easy to game under imbalance. | reported per bucket and per criterion |
| **precision** | of the things you *called* X, how many were X? | inside F1 |
| **recall** | of the things that *were* X, how many did you catch? | inside F1 |
| **F1** | the harmonic mean of precision and recall — one number that punishes being lopsided | the headline label metric |
| **macro-F1** | F1 computed per class then averaged **equally** | so the rare `insufficient_evidence` class counts as much as `met`. Capped at (classes present)/3 — the 0.67 trap |
| **confusion matrix** | grid of what was predicted versus what was true | shows *which* label it confuses with which |
| **baseline** | the simple thing your complicated thing must beat | row 0 — and it essentially ties |
| **ablation** | remove or change **one** component at a time to attribute effect to cause | the six configurations |
| **confidence interval** | the range the true value plausibly sits in | every number in the results table |
| **Wilson interval** | a CI for a proportion that stays sane at small n and near 0 or 1 | all the proportion columns |
| **bootstrap** | resample your data many times and watch how much the answer moves | the F1 intervals |
| **cluster bootstrap** | resample **groups** rather than rows, when rows within a group are correlated | resamples the 40 *patients*, not the 239 decisions |
| **calibration** | does stated confidence match actual correctness? | measured: gap of **+0.009**, i.e. none |
| **risk–coverage** | as you answer fewer questions, does your error rate on the rest fall? | the abstention argument, plotted |
| **abstention** | declining to answer rather than guessing | triggered by an unverified quote |
| **provenance** | *where did this text come from?* | what string matching proves |
| **entailment** | *does this text actually support that claim?* | what string matching **cannot** prove — the acknowledged gap |
| **determinism** | same input, same output, every run | seeded RNG, temperature 0, cached responses |
| **sensitivity analysis** | re-run the headline number under a different defensible assumption and report both | the B1/B2 alternative-route disagreement |

---

## 11. Questions you should be able to answer

**Why not just put the whole policy in the prompt?**
You can, and it is row 0 — F1 0.86, statistically indistinguishable from the full pipeline. It also
burns far more tokens per case, does not scale past one policy, and in this run it fabricated 13
citations where the retrieval configs fabricated none. Retrieval is about scale and citation
quality, not raw accuracy on one document.

**Why is quote checking string matching rather than a model?**
Because then "are the quotes real?" inherits the second model's errors and the number becomes
arguable. `in` is not arguable.

**How do you know the answer key is right?**
It is not validated externally — that is a stated limitation. What is controlled is that it was
frozen before any model output was seen, it is 60 lines of if-statements anyone can read, and the
one place it disagrees with the model is reported both ways rather than silently resolved.

**Why did abstention not help?**
Because after the retrieval and prompt fixes there was nothing left to catch, and the decisions it
*would* have caught were disproportionately correct. The honest claim is narrower than the original
hypothesis: verification buys citation integrity, not accuracy.

**What would you do next?**
Write the ten handwritten cases and see whether the numbers hold; work out why BM25 hurts; then
entailment checking — which is the one thing that would turn "the quote is real" into "the quote
supports the decision".
