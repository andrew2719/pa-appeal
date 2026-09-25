# Documents to download by hand

Save each page as HTML into `data/raw/`, then convert in notebook 01.
**No scraping.** Noridian and CGS host the same documents as cms.gov and save more cleanly.

All 13 were downloaded by hand on **2026-09-22**. Revision numbers are the ones shown on the
page at that date — coverage rules change, so any number in the README belongs to these versions
and not to whatever is current when you read this.

## The ones that matter

| ID | What it is | Revision at download | got it |
|---|---|---|---|
| L33718 | PAP devices for sleep apnea — **the whole project is about this one** | R11, effective 01/01/2024 | ☑ |
| A52467 | PAP coding article | R15, effective 08/08/2021 | ☑ |
| NCD 240.4 | CPAP for OSA | version effective 03/13/2008 | ☑ |
| NCD 240.4.1 | Sleep testing for OSA | version effective 03/03/2009 | ☑ |
| A55426 | Standard documentation requirements | R28, effective 01/01/2024 | ☑ |

## Decoys — 8 unrelated equipment policies

Without these, "did it find the right document?" is always yes and that column of my table means
nothing.

| # | Topic | LCD ID | got it |
|---|---|---|---|
| 1 | Oxygen and oxygen equipment | L33797 | ☑ |
| 2 | Hospital beds | L33820 | ☑ |
| 3 | Manual wheelchairs | L33788 | ☑ |
| 4 | Power mobility devices | L33789 | ☑ |
| 5 | Nebulizers | L33370 | ☑ |
| 6 | Glucose monitors | L33822 | ☑ |
| 7 | Surgical dressings | L33831 | ☑ |
| 8 | External infusion pumps | L33794 | ☑ |

Skip respiratory assist devices — that policy covers E0470/E0471 and overlaps the real rules, so
it's a hard case rather than a decoy. Interesting to add later, muddies the number now.

## How they are used

`01_policy.ipynb` converts each HTML file to markdown in `data/policies/`. Notebook 03 indexes
L33718 one chunk per criterion and splits the other twelve on their headings, so the decoys are
genuinely in the retrieval pool.

Notebook 05 sorts retrieved documents three ways, which matters when reading the results:

- **target** — L33718, the policy that governs the case
- **supporting** — A52467, A55426, NCD 240.4, NCD 240.4.1. Retrieving these is reasonable; they
  are the coding article, the documentation requirements, and the two national determinations.
- **decoy** — the eight equipment LCDs above. Only these tell you whether retrieval is being
  fooled.

In the 40-case run the retrieved context was 60% target, 25% supporting, 15% decoy.

## One caveat about the decoys

The Standard Written Order sentence behind the `swo` criterion is DME boilerplate and appears
**verbatim in seven of the eight decoy policies**. Retrieval rank for that criterion is therefore
measured by chunk label rather than by text match — matching on text would happily score a decoy
chunk as a hit. Worth remembering before reading anything into `swo`'s retrieval numbers.
