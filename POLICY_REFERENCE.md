# Medicare Policy and Project Code Reference

This guide explains the Medicare policy documents used by the project, the PAP/CPAP rules used in the toy notebook, and the labels created by our experiment.

> This is a study and software-project reference, not medical or legal advice. Medicare policies can change. Before making a real coverage or appeal decision, check the current CMS Medicare Coverage Database and the policy version that applied on the claim's date of service.

## 1. The basic idea

Medicare does not cover an item merely because a clinician prescribed it. A claim normally has to answer several different questions:

1. **Is the item part of a Medicare benefit category?**
2. **Is it reasonable and necessary for this patient's condition?**
3. **Does the medical record contain the required evidence?**
4. **Were ordering, timing, supplier, and billing requirements followed?**
5. **Was the correct HCPCS billing code used?**

Our project turns policy language into smaller machine-checkable criteria and asks a model whether a patient record supplies enough evidence for each criterion.

## 2. The kinds of CMS documents

### NCD — National Coverage Determination

An NCD is a national Medicare coverage rule. It applies across the country.

Examples in this project:

- **NCD 240.4** — CPAP therapy for obstructive sleep apnea.
- **NCD 240.4.1** — sleep testing used to diagnose obstructive sleep apnea.

Think of an NCD as the nationwide coverage foundation.

### LCD — Local Coverage Determination

An LCD is issued by a Medicare Administrative Contractor, or MAC. It explains when a service or item is considered reasonable and necessary in the contractor's jurisdiction.

The main CPAP document in this project is:

- **L33718** — Positive Airway Pressure devices for treatment of obstructive sleep apnea.

The `L` in `L33718` means that this document is an LCD. The remaining digits identify the document; they are not a medical measurement or billing code.

### Policy Article

A related Policy Article supplies coding, payment, and documentation details that support an LCD.

The main CPAP article in this project is:

- **A52467** — the Policy Article related to PAP devices for obstructive sleep apnea.
- **A55426** — standard documentation requirements for claims submitted to DME MACs.

The `A` means article. An LCD and its related article should be read together: the LCD mainly describes medical-necessity coverage rules, while the article adds operational, coding, and documentation guidance.

## 3. The 13 documents in this project

| Document | Subject | Why it may matter |
| --- | --- | --- |
| `L33718` | PAP devices for obstructive sleep apnea | Main policy used by the toy experiment |
| `A52467` | PAP Policy Article | PAP coding, payment, and documentation details |
| `A55426` | Standard DME claim documentation | General order and medical-record requirements |
| `NCD240.4` | CPAP therapy for OSA | National CPAP coverage foundation |
| `NCD240.4.1` | Sleep testing for OSA | Rules for a qualifying diagnostic sleep test |
| `L33370` | Nebulizers | Another DME policy domain |
| `L33788` | Manual wheelchair bases | Another DME policy domain |
| `L33789` | Power mobility devices | Another DME policy domain |
| `L33794` | External infusion pumps | Another DME policy domain |
| `L33797` | Oxygen and oxygen equipment | Can interact with PAP when oxygen is used concurrently |
| `L33820` | Hospital beds and accessories | Another DME policy domain |
| `L33822` | Glucose monitors | Another DME policy domain |
| `L33831` | Surgical dressings | Another DME policy domain |

The non-CPAP policies let the project grow beyond one policy after the CPAP prototype works.

## 4. Important PAP and sleep terms

| Term | Plain-language meaning |
| --- | --- |
| **OSA** | Obstructive sleep apnea: repeated blockage or collapse of the airway during sleep |
| **PAP** | Positive airway pressure; the broad family of devices used to keep the airway open |
| **CPAP** | Continuous PAP; supplies one continuous pressure level |
| **Bi-level PAP** | Supplies different inspiratory and expiratory pressure levels |
| **AHI** | Apnea-hypopnea index: average apnea and hypopnea events per hour of sleep |
| **RDI** | Respiratory disturbance index used by the policy for certain sleep studies; calculated per hour of recording |
| **DME** | Durable medical equipment |
| **DMEPOS** | Durable medical equipment, prosthetics, orthotics, and supplies |
| **HCPCS code** | A standardized billing code identifying an item or service |
| **Beneficiary** | The person receiving Medicare benefits; effectively the patient in our examples |
| **Treating practitioner** | The qualified clinician responsible for evaluating or treating the patient |

## 5. The PAP device codes

These are real HCPCS billing codes—not labels invented by the project.

| Code | Device | Simplified policy meaning for OSA |
| --- | --- | --- |
| `E0601` | Single-level CPAP device | May be covered when the initial A–C requirements are met |
| `E0470` | Bi-level respiratory-assist device **without** backup rate | May be covered for OSA when A–C are met and E0601 was tried and proved ineffective under the policy definition |
| `E0471` | Bi-level respiratory-assist device **with** backup rate | L33718 says it is not reasonable and necessary when the primary diagnosis is OSA |

`E0470` and `E0471` are not interchangeable. The presence or absence of a backup rate is important.

## 6. The simplified initial-coverage structure

L33718 says an `E0601` device is covered for OSA when criteria A through C are met.

### A — Evaluation before the sleep test

The patient must have an in-person clinical evaluation by the treating practitioner before the sleep test to assess for OSA.

### B — A qualifying sleep test

The sleep test must meet either route B1 or route B2:

#### B1 — Higher AHI/RDI route

- AHI or RDI is at least 15 events per hour; and
- the test has the policy-required minimum of 30 events.

#### B2 — Lower AHI/RDI plus symptoms/conditions route

- AHI or RDI is from 5 through 14 events per hour;
- the test has the policy-required minimum of 10 events; and
- the record documents at least one listed symptom or condition.

The listed evidence includes excessive daytime sleepiness, impaired cognition, a mood disorder, insomnia, hypertension, ischemic heart disease, or a history of stroke.

### C — Instruction in using the equipment

The patient or caregiver must receive supplier instruction on proper use and care of the equipment.

### D — Additional requirement for E0470

For `E0470`, the A–C requirements still apply, and an `E0601` must also have been tried and proved ineffective. The policy defines ineffective more narrowly than simply saying that the patient disliked CPAP: the record must document failure to meet therapeutic goals despite appropriate mask fitting and pressure settings.

## 7. Continued coverage after the initial trial

Initial qualification is not the end of the policy. Continued PAP coverage beyond the first three months generally requires:

- a treating-practitioner re-evaluation no sooner than day 31 and no later than day 91;
- documentation that OSA symptoms improved; and
- objective adherence evidence reviewed by the practitioner.

The policy defines adherence as PAP use for at least 4 hours per night on 70% of nights during a consecutive 30-day period within the first three months.

These are different questions from initial sleep-test qualification. A patient may qualify initially but lack the evidence required for continued coverage.

## 8. What `B1`, `B2`, and `E0471_osa` mean in our project

There are two kinds of identifiers in the notebook:

1. **Real CMS codes**, such as `E0601`, `E0470`, and `E0471`.
2. **Our internal criterion IDs**, such as `B1`, `B2`, and `E0471_osa`.

`B1` and `B2` are convenient names based on the two alternatives under criterion B in L33718. They are not HCPCS billing codes.

`E0471_osa` is an internal, readable identifier for the policy rule involving HCPCS code `E0471` when the primary diagnosis is OSA. The `E0471` portion is a real code; the `_osa` suffix was added by the project.

An important interpretation rule:

> If `E0471_osa` is labeled `met`, it can mean that the **noncoverage rule applies**. It does not necessarily mean that E0471 is approved or covered.

Always read the criterion text in `criteria.json` before interpreting `met`. A model is deciding whether the criterion's statement applies—not whether the final claim should automatically be paid.

## 9. What `yes`, `no`, and `silent` mean

These are synthetic case names from `00_toy.ipynb`. They are not CMS terms.

| Toy case | What we intentionally put in the record | Expected label for the tested patient criterion |
| --- | --- | --- |
| `yes` | Evidence that supports the criterion | `met` |
| `no` | Evidence that directly contradicts or fails the criterion | `unmet` |
| `silent` | The record does not say enough either way | `insufficient_evidence` |

The `silent` case is especially important. Missing documentation is not the same thing as evidence that something did not happen.

For example:

- “AHI is 18” can support a threshold criterion.
- “AHI is 3” can contradict that threshold criterion.
- No AHI anywhere in the supplied record means the result should normally be `insufficient_evidence`, not an invented value and not automatically `unmet`.

## 10. The three decision labels

### `met`

The supplied record contains enough relevant evidence to satisfy the criterion as written.

### `unmet`

The supplied record contains evidence showing that the criterion is not satisfied.

### `insufficient_evidence`

The available record does not contain enough information to decide. This is not a softer spelling of `unmet`; it is a separate and valuable outcome.

## 11. Understanding the toy output

The toy run produced the intended distinction for `B1`:

```text
yes     -> met
no      -> unmet
silent  -> insufficient_evidence
```

That shows the model can distinguish supporting evidence, contradictory evidence, and missing evidence.

The output also showed:

```text
quote in policy: True
```

This means the returned policy quotation was found verbatim in the policy text. It is a useful anti-hallucination check, but it does **not** by itself prove that:

- the classification is correct;
- the quotation applies to the patient;
- patient evidence exists; or
- the final claim should be approved or denied.

Later evaluation should verify policy evidence and patient-record evidence separately.

## 12. How the files fit together

```text
data/raw/*.html
    Original CMS pages

data/policies/*.md
    Clean policy text used by the model

data/criteria.json
    Small, named rules extracted from the policy

Toy or evaluation patient records
    Evidence supplied for an individual case

Model decision
    met / unmet / insufficient_evidence, with evidence and reasoning

Validation and results
    Checks quotes, expected labels, errors, and experiment performance
```

The policy tells us **what must be shown**. The patient record tells us **what was actually documented**. `criteria.json` connects those two sides.

## 13. A quick checklist for when you get lost

When reading an output, ask these questions in order:

1. **Which document is this?** NCD, LCD, or article?
2. **Is the identifier a real HCPCS code or our internal criterion ID?**
3. **Does the criterion describe coverage, noncoverage, documentation, or timing?**
4. **Is this initial coverage or continued coverage?**
5. **What exact policy language supports the rule?**
6. **What exact patient-record language supports or contradicts it?**
7. **If the record is silent, did the system correctly use `insufficient_evidence`?**
8. **Does `met` mean a coverage requirement was met, or that a denial/exclusion rule applies?**

## 14. Official references

- [CMS LCD L33718 — PAP Devices for OSA](https://www.cms.gov/medicare-coverage-database/view/lcd.aspx?LCDId=33718)
- [CMS Policy Article A52467 — PAP Devices for OSA](https://www.cms.gov/medicare-coverage-database/view/article.aspx?articleId=52467)
- [CMS NCD 240.4 — CPAP Therapy for OSA](https://www.cms.gov/medicare-coverage-database/view/ncd.aspx?ncdid=226)
- [CMS Medicare Coverage Database search](https://www.cms.gov/medicare-coverage-database/search.aspx)

When the wording in this guide and a current CMS page differs, use the official policy version applicable to the claim's date of service.


Let’s pause the coding and build the mental model first. There are several different kinds of “codes” in this project, and mixing them together makes everything feel much harder than it is.

## The whole project in one sentence

We are testing whether a machine can:

> Read a patient’s documents, retrieve the correct Medicare rule, and correctly identify which coverage requirements are satisfied, failed, or undocumented.

The machine is not diagnosing sleep apnea. It is checking whether the documentation satisfies Medicare coverage rules.

## 1. The different identifiers

| Example     | What it represents                      |
| ----------- | --------------------------------------- |
| `L33718`    | A Medicare policy document              |
| `NCD 240.4` | A national Medicare determination       |
| `A52467`    | A supporting policy article             |
| `E0601`     | A billable CPAP device code             |
| `E0470`     | A billable bi-level device code         |
| `B1`        | Our ID for one L33718 requirement       |
| `case_001`  | A fictional patient case                |
| `clear_met` | A group—or bucket—of similar test cases |

These identifiers belong to different layers. For example, `L33718` is not a machine, while `E0601` is a code representing a machine.

## 2. What the devices are

### `E0601`

A normal single-level CPAP device.

It provides one continuous level of positive airway pressure.

For initial Medicare coverage under L33718, the patient generally needs:

* Evaluation before the sleep test
* A qualifying sleep test through B1 or B2
* Instruction on using and caring for the equipment
* A valid order and other documentation

### `E0470`

A bi-level device without a backup rate.

It uses different pressures for breathing in and breathing out.

For OSA coverage, the patient must satisfy the regular E0601 requirements and must also have tried E0601 and found it ineffective under the policy definition.

### `E0471`

A bi-level device with a backup rate.

L33718 says E0471 is not reasonable and necessary when the primary diagnosis is OSA. ([www.cms.gov][1])

That does not mean E0471 is never medically useful. It means this particular OSA policy does not cover it for that situation. Other diagnoses are handled by other policies.

## 3. What the sleep test measures

### Apnea

A period when airflow stops for at least ten seconds.

### Hypopnea

A partial reduction in airflow lasting at least ten seconds and meeting the policy’s airflow and oxygen-desaturation requirements.

### AHI

Apnea-hypopnea index:

> Average apnea and hypopnea events per hour of actual sleep.

### RDI

Respiratory disturbance index:

> Average apnea and hypopnea events per hour of recording.

The distinction matters because different sleep-study types measure sleep and recording time differently.

## 4. The sleep-test types

L33718 recognizes several sleep-test categories:

* Type I
* Type II
* Type III
* Type IV
* Other qualifying home sleep tests

We are not building or testing these instruments. Our fictional patient record simply states which test was performed.

For a sleep test to support PAP coverage, the policy checks more than its type:

* Is it Medicare-valid?
* Was an FDA-approved diagnostic device used?
* Did it meet the requirements in effect on the claim date?
* Was it ordered by the treating practitioner?
* Was it conducted by a qualified sleep-test provider?
* Were applicable state requirements followed?

Those checks are grouped under:

```text
sleep_test_valid
```

## 5. What is a criterion?

A criterion is one small policy rule that the system can check independently.

For example:

```text
B1
```

asks:

> Does the record show AHI or RDI of at least 15 events per hour and at least 30 recorded events?

The model assigns one of three labels:

| Label                   | Meaning                                                                     |
| ----------------------- | --------------------------------------------------------------------------- |
| `met`                   | The record contains enough evidence that the requirement is satisfied       |
| `unmet`                 | The record contains evidence showing it is not satisfied                    |
| `insufficient_evidence` | The requirement applies, but the record does not provide enough information |

`criteria.json` is therefore a machine-readable version of the important parts of L33718.

## 6. What is inside `criteria.json`?

Your 20 criteria fall into four groups.

### Definitions: five entries

These teach the model what policy terminology means:

```text
apnea_def
hypopnea_def
ahi_def
rdi_def
pap_device_codes
```

They are used during retrieval but are not patient pass/fail questions.

We do not want output such as:

```text
ahi_def = met
```

A definition cannot be met by a patient. It only explains terminology.

### Initial coverage: nine entries

These concern obtaining the device initially.

#### `initial_evaluation`

Was there an in-person clinical evaluation before the sleep test?

#### `B1`

Was AHI/RDI at least 15 with at least 30 events?

#### `B2`

Was AHI/RDI between 5 and 14, with at least 10 events and a qualifying symptom or condition?

Qualifying evidence can include:

* Excessive daytime sleepiness
* Impaired cognition
* Mood disorder
* Insomnia
* Hypertension
* Ischemic heart disease
* History of stroke

B1 and B2 are alternative routes:

```text
B1 OR B2
```

The patient does not need both.

#### `short_study_events`

For studies shorter than two hours, was the required minimum number of events still recorded?

#### `sleep_test_valid`

Was the sleep test valid, approved, properly ordered, and properly conducted?

#### `device_instruction`

Did the supplier teach the patient or caregiver how to use and care for the device?

#### `E0470_trial`

Was E0601 tried and proven ineffective before requesting E0470?

#### `E0470_ineffective`

Does the record demonstrate what the policy means by “ineffective”—failure to reach therapeutic goals despite appropriate mask fitting and pressure settings?

#### `E0471_osa`

Is the device eligible under this OSA policy rather than being an E0471 requested for OSA?

### Continued coverage: five entries

These concern coverage after the initial trial.

#### `reeval_window`

Did the clinical reevaluation occur between days 31 and 91?

#### `reeval_late`

If it occurred after day 91, did the late-evaluation rule allow coverage to restart from the evaluation date?

#### `symptoms_improved`

Did the practitioner document improvement in OSA symptoms?

#### `adherence_reviewed`

Did the practitioner review objective adherence information?

#### `adherence`

Does the usage data show:

* At least four hours per night
* On at least 70% of nights
* During a consecutive 30-day period
* Within the initial three months?

These continued-coverage requirements come directly from L33718. ([www.cms.gov][1])

### General requirement: one entry

#### `swo`

Was a completed Standard Written Order communicated to the supplier before submitting the claim?

## 7. Why are some criteria omitted?

“Omitted” has a different meaning from `insufficient_evidence`.

### Omitted means not applicable

Example: an E0601 patient does not need the E0470 trial criterion.

Therefore:

```text
E0470_trial
```

is omitted entirely.

### `insufficient_evidence` means applicable but missing

Example: an E0470 patient needs evidence that E0601 was tried, but the record says nothing about it.

Then:

```text
E0470_trial = insufficient_evidence
```

### `unmet` means evidence of failure

Example: the record explicitly says E0601 was never tried.

Then:

```text
E0470_trial = unmet
```

That distinction is fundamental:

| Situation                                | Result                  |
| ---------------------------------------- | ----------------------- |
| Rule does not apply                      | Omit it                 |
| Rule applies, but information is absent  | `insufficient_evidence` |
| Rule applies and evidence contradicts it | `unmet`                 |
| Rule applies and evidence supports it    | `met`                   |

## 8. Examples of conditional omission

### `short_study_events`

This rule matters only when the sleep study lasted less than two hours.

For a three-hour study:

```text
short_study_events → omitted
```

It is not missing information. The special rule simply does not apply.

### `reeval_late`

This matters only when reevaluation happened after day 91.

For a day-60 evaluation:

```text
reeval_late → omitted
```

The normal `reeval_window` criterion handles that patient.

### Device-specific requirements

For `E0601` initial coverage:

```text
E0470_trial → omitted
E0470_ineffective → omitted
E0471_osa → omitted
```

For `E0470` initial coverage:

```text
E0470_trial → checked
E0470_ineffective → checked
E0471_osa → omitted
```

For an `E0471` OSA request:

```text
E0471_osa → checked
```

## 9. What are the buckets?

Buckets are not Medicare concepts. We invented them to create a balanced test dataset.

```python
BUCKETS = {
    "clear_met": 8,
    "met_5_14": 6,
    "clear_unmet": 6,
    "insufficient": 8,
    "borderline": 8,
    "adversarial": 4,
}
```

Together, they create 40 fictional patients.

### `clear_met`

Easy positive cases.

Example:

```text
AHI = 22
events = 46
evaluation completed
valid sleep test
instruction documented
```

Expected B1 result:

```text
met
```

### `met_5_14`

Cases qualifying through B2 rather than B1.

Example:

```text
AHI = 11
events = 22
hypertension documented
```

Expected:

```text
B1 = unmet
B2 = met
```

The patient can still qualify because B1 and B2 are alternatives.

### `clear_unmet`

Clear failures.

Examples:

```text
AHI = 3
```

or:

```text
E0471 requested with OSA as the primary diagnosis
```

### `insufficient`

Important information is absent.

Examples:

```text
AHI not documented
event count missing
adherence report unavailable
```

The correct result is usually `insufficient_evidence`, not `unmet`.

### `borderline`

Cases near important thresholds:

```text
AHI = 14
AHI = 15 but only 25 events
68% adherence
reevaluation on day 95
```

These test whether the model handles exact numerical boundaries.

### `adversarial`

Intentionally confusing cases.

Examples:

* The denial letter quotes the wrong policy rule.
* The chart contains a similar-looking but irrelevant number.
* A sentence says “four hours recommended” but does not document actual usage.
* The denial letter says the patient failed while the sleep study proves otherwise.

These test whether the model trusts the actual evidence rather than blindly copying the denial letter.

## 10. Why did we download the other policies?

The five PAP-related documents are:

```text
L33718
A52467
A55426
NCD240.4
NCD240.4.1
```

The other eight are unrelated DME policies, such as wheelchairs, oxygen, hospital beds, and surgical dressings.

They are not useless. They act as **retrieval decoys**.

When the pipeline searches all documents for a CPAP question, it should retrieve L33718 or its related PAP documents—not a wheelchair or nebulizer policy.

That lets us measure two different abilities:

1. Did retrieval find the correct policy?
2. Given the correct policy, did the model interpret the patient evidence correctly?

For this first experiment, `criteria.json` contains only L33718 criteria. Later, the project could add criteria from A52467, A55426, and other policies.

## 11. How the case system works

Each fictional patient begins as a structured `spec`:

```python
spec = {
    "device": "E0601",
    "phase": "initial",
    "ahi": 18,
    "events": 38,
    "initial_eval_before_test": True,
    "swo_on_file": None,
}
```

Two separate functions use it:

### `oracle(spec)`

Produces the correct answer:

```python
{
    "B1": "met",
    "B2": "unmet",
    "initial_evaluation": "met",
    "swo": "insufficient_evidence",
}
```

### `render(spec)`

Turns the same facts into human-looking medical documents:

```text
Sleep study demonstrated an AHI of 18 events per hour,
with 38 respiratory events recorded.
```

The model sees only the rendered documents. It never sees the structured `spec` or oracle answer.

Finally, we compare:

```text
Model prediction versus oracle answer
```

That is the experiment. The oracle represents truth; the rendered documents represent what the model must read and understand.

[1]: https://www.cms.gov/medicare-coverage-database/view/lcd.aspx?LCDId=33718&utm_source=chatgpt.com "LCD - Positive Airway Pressure (PAP) Devices for the Treatment of Obstructive Sleep Apnea (L33718)"
