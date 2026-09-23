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
