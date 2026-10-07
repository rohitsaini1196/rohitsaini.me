---
title: "How I Reconstructed Bengaluru's Public Works From Fragmented Government Data"
date: 2026-10-07
slug: bengaluru-public-works
readTime: 6
draft: false
tags: ["build", "backend", "systems"]
description: "Reconstructing 432 public works projects and ₹544 crore in BBMP payments across messy PDFs, changing ward maps, and OCR arithmetic checks."
---

I wanted to answer one simple question:

> If BBMP spends money on a road, park, drain or footpath near me, can I reconstruct what actually happened?

Not just the tender. The full lifecycle:

```text
planned
  ↓
contracted
  ↓
billed
  ↓
paid
  ↓
completed
  ↓
evidence
```

So I picked Koramangala and started building.

A few iterations later, the system had reconstructed **432 public works projects**, **592 paid bills** and roughly **₹544 crore in gross payments** from public BBMP and Karnataka procurement records.

**Live:** [bengaluru-public-works](https://rohitsaini1196.github.io/bengaluru-public-works/) · **Code:** [github.com/rohitsaini1196/bengaluru-public-works](https://github.com/rohitsaini1196/bengaluru-public-works)

The interesting part was not the scraping. It was making the data trustworthy.

Everything came from public endpoints, rate-limited to about one request per second and cached. Where something needed a login, I stopped.

---

## The real problem was entity resolution

Government data is surprisingly rich. It is also fragmented.

The same project may appear across:

- BBMP payment systems
- KPPP procurement records
- archived bill registers
- work orders
- agreements
- Schedule B documents
- bill forms
- completion certificates
- extension-of-time orders
- scanned images

And these systems often do not share a stable identifier:

- A tender number might not appear in the bill system.
- A job number may be reused under a different ward map.
- A contractor name may have multiple spellings.
- A project description may change slightly between systems.

So the first real engineering problem became:

> **How do I know two records refer to the same physical work?**

The pipeline scores multiple signals:

```text
description similarity
+ locality
+ ward mapping
+ contractor
+ dates
+ work type
+ amounts
```

No single fuzzy match is treated as truth.

That turned out to be important. At one point, several projects appeared to have been paid 2x–3x their contract value. The cause wasn't misconduct. It was a bug: **job-number collisions across Bengaluru's changing ward maps**. Job 186-23-000001, for example, exists in both Koramangala and Jaraganahalli.

Same-looking identifier, different work. The system had merged them. Once fixed, several major anomalies disappeared.

---

## Provenance became a first-class data model

I did not want the output to become:

```text
contract_value = 1.4 crore
```

I wanted:

```text
contract_value = 1.4 crore

source:
  work_order.pdf
  page: 3

method:
  OCR

confidence:
  inferred
```

Every fact is classified as `confirmed`, `inferred` or `unknown`, and every fact carries provenance.

This changes how you build the rest of the system. If two sources disagree, you do not silently overwrite one. If OCR produces a number, it does not outrank a direct IFMS value.

The precedence is roughly:

```text
direct API
    >
native document text
    >
OCR validated by arithmetic
    >
unvalidated OCR (never used for quantities)
    >
handwritten / unreadable (never read)
```

That sounds obvious. It becomes less obvious once you have thousands of facts coming from hundreds of ugly PDFs.

---

## OCR alone was not good enough

A lot of the useful data lives inside scanned government documents, and OCR makes hilarious mistakes:

```text
2,00  →  200
```

or:

```text
quantity ↔ rate
```

or decimal points disappear entirely.

So I stopped trusting OCR text directly. For quantity rows, the parser only accepts values when:

```text
quantity × rate ≈ amount
```

That gives an independent arithmetic check. A real row from a comparative statement (the official tender-vs-executed reconciliation):

```text
tendered:   5,354.95 sqm × ₹157.30 = ₹8,42,333.64
executed:     730.20 sqm × ₹157.30 = ₹1,14,860.46
savings:                             ₹7,27,473.18   (= tender − executed, exactly)
```

Three numbers that reconcile three ways. If they do not reconcile, the row is rejected.

This became especially useful for reading:

- Schedule B
- bill forms
- comparative statements
- work slips

The parser also learned to handle:

- rotated and upside-down documents
- table lines interfering with OCR
- swapped columns
- decimal restoration
- duplicate re-uploads
- cumulative "up-to-date" quantities in running bills

That last one mattered more than I expected. On one road project, a running bill showed an item at exactly its Schedule B quantity: 1,205.16 cubic metres. The final comparative statement said **1,771.56** executed, split at a lower rate above 125%. Reading the wrong column, or the wrong bill, turns "+47%" into "identical to the estimate".

The handwritten measurement books are still mostly hopeless: only 8 of the 144 I downloaded yielded a checked number. Typed bill forms turned out to be much more useful.

---

## Then I made the system generate red flags

Once the lifecycle was reconstructed, it became possible to ask simple rule-based questions:

```text
payment > 125% of contract?
```

```text
billed quantity > 125% of Schedule B?
```

```text
project past completion date with no final bill?
```

```text
same document attached to multiple jobs?
```

```text
final bill but no visible measurement evidence?
```

This worked. Too well. The first version produced some very dramatic-looking cases.

That was the next engineering problem.

---

## A signal is not a finding

The system initially found cases like:

- a park paid at **220% of contract value**
- outdoor gym equipment billed at **2x the tendered quantity**
- identical documents under different projects
- final bills with no measurement books
- quantities matching Schedule B exactly

They looked bad. A lot of them were wrong.

So I added a second pipeline:

```text
signal
  ↓
try to disprove it
  ↓
fetch more documents
  ↓
look for variations / work slips / extensions
  ↓
recalculate
  ↓
keep or kill the signal
```

This was probably the most important architectural change. A few examples follow.

### Paid 220% of contract

Wrong. There were two work orders under the same job, ₹3.51 crore and ₹3.00 crore. The first one was scanned upside down and had been missed. Against both contracts, payments come to about 101%.

### Gym equipment billed above tender quantity

Also explained. There was an approved work slip recording five machines executed per item. The file was named with `WS`, which the earlier document classifier did not recognise.

### Same contract under two jobs

It turned out to be one package contract covering 13 works, booked under two job codes. Payments across both jobs come to 99.8% of the contract.

### Final bills with no measurement books

For bills from about mid-2024 onwards, the public IFMS endpoint doesn't expose bill-level attachments without a login: none of the 117 such bills I checked returned any. Absence from the public response was not evidence that the document did not exist. The two remaining cases had their MB filed under a different attachment type.

---

## 23 signals went in. 5 survived.

By the final review stage, the system had examined **23 candidate cases**:

- **7** were explained by documents
- **6** were artefacts of my own pipeline
- **3** rested on evidence too weak to present
- **2** had no real issue once examined
- **5** survived as investigation-ready questions

That was a better outcome than finding 23 "suspicious" projects, because the system had become capable of this:

```text
interesting anomaly
      ↓
better evidence
      ↓
"never mind, this is explained"
```

That is exactly what I wanted.

---

## What survived

None of these is a finding of wrongdoing. Each is a question the public record can't answer, published with its legitimate explanations and the exact record that would settle it.

- **The Ejipura–Sony World elevated corridor.**
  - ₹160.7 crore paid, 30% of all the money in the dataset.
  - The original contract was extended twice, to December 2020 and then December 2021.
  - The balance work was re-awarded with a completion date of February 2025. No extension for that date is visible, there is no final bill, and ₹15 crore of bills are pending.
  - Penalties written into the extension orders total ₹1.58 crore; ₹31 lakh appears as fines actually deducted.
- **A ₹21-crore road package.** Four items' final quantities match the tender to the decimal. They sit on a "final" bill of **₹1**, registered in 2019 and never paid.
- **A street-light maintenance contract** for eight months, billed for fifteen, at 226% of the winning bid.

Each could have a perfectly ordinary explanation: an extension that was never uploaded, a renewal order, a closing entry. The point is that you can now ask for that specific document.

---

## The system is closer to a debugger than a fraud detector

The current architecture looks roughly like this:

```text
BBMP IFMS ───────┐
KPPP ────────────┼──► discovery
OpenCity ────────┘
                       ↓
                 scope + matching
                       ↓
                  truth model
                       ↓
                 provenance layer
                       ↓
               quantity extraction
                       ↓
                    signals
                       ↓
               adversarial review
                       ↓
             investigation cases
                       ↓
                 public explorer
```

The final output is not:

> This project is wrong.

It is:

> This value looks inconsistent with the public records we found.

Then:

> Here are the documents.

Then:

> Here are plausible explanations.

Then:

> Here is the exact missing record that would resolve the question.

That feels much more useful.

---

## Current result

The public version currently covers Koramangala. It includes:

- **432 reconstructed projects**
- **388 projects with BBMP bills**
- **592 paid bills**
- **₹543.99 crore in gross payments**
- quantity extraction from Schedule B and bill forms
- provenance for individual facts
- investigation case packs, each with the specific records to request
- a read-only public explorer

The pipeline is also configurable by area, so Koramangala is not hard-coded into the architecture. A test configuration for HSR Layout found 549 paid bills in the same city-wide payment data with zero code changes.

The next obvious experiment is doing that properly for another Bengaluru locality. Not because I need more data, but because I want to know whether the abstraction actually generalises.

---

## What still does not work well

Some problems remain genuinely hard.

### Handwritten measurement books

OCR is nowhere near reliable enough.

### Physical verification

Administrative documents do not prove that a road exists, or that it was built to the claimed quality.

### GPS

None of the site photos I checked carried a GPS location.

### Missing public documents

Sometimes the right conclusion is simply:

```text
unknown
```

And I think that is fine. A system like this becomes dangerous the moment it starts treating missing data as proof.

---

## It is open source

Both the code and the public explorer are available.

**Live:** [bengaluru-public-works](https://rohitsaini1196.github.io/bengaluru-public-works/)  
**Code:** [github.com/rohitsaini1196/bengaluru-public-works](https://github.com/rohitsaini1196/bengaluru-public-works)

The code (MIT) includes:

- ingestion
- entity resolution
- truth/provenance model
- OCR pipeline
- quantity parsing
- signal generation
- case review
- public export

The data is CC BY 4.0. If a document explains one of the open cases, tell me and it will be closed publicly.

The part I find most interesting is not that it can find anomalies. It is that it can increasingly **explain them away**.

That feels like a much better direction for building systems around messy public data.
