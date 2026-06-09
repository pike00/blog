---
title: "Daraxonrasib and the RAS(ON) bet"
description: "A software person reads two NEJM papers on daraxonrasib — a RAS(ON) multi-selective inhibitor that nearly doubled overall survival in previously treated metastatic pancreatic cancer — and why drugging RAS in its active state, not just KRAS G12C, is the shift that matters."
date: "2026-06-08"
tags: ["Biology", "Cancer", "Oncology", "Science"]
draft: true
---

I am not an oncologist. I write software and run a homelab. But every so often a result lands outside my lane that is large enough to be legible to an outsider, and this is one: a once-daily pill that roughly doubled overall survival in second-line metastatic pancreatic cancer — a setting where the standard of care is measured in a handful of months — while being *easier* to tolerate than the chemotherapy it beat.

The drug is **daraxonrasib** (development name RMC-6236, from Revolution Medicines). The phase 3 trial, RASolute 302, was presented at the 2026 ASCO plenary and published in *NEJM* alongside the earlier phase 1/2 data. What follows is my attempt to understand why it works, told from the perspective of someone who had to look up what a GTPase is. The short version: the interesting part is not the survival number. It is *which state of the protein the drug grabs*.

## Why pancreatic cancer is the right place to be impressed

Pancreatic ductal adenocarcinoma (PDAC) is one of the cancers that has stubbornly refused to improve. Once it is metastatic and has progressed past first-line chemotherapy, the options are thin and the prognosis is brutal — historically a median overall survival on the order of six months. For decades, second-line PDAC has been where promising drugs go to post a flat survival curve.

The reason it is so hard is genetic and almost monotonous: roughly **90% of pancreatic cancers carry a mutation in *KRAS***. It is not one of several drivers; it is *the* driver, present in the overwhelming majority of tumors. And the mutations cluster tightly at a single amino acid — codon 12:

| KRAS mutation in PDAC | Approx. share |
|---|---|
| G12D (glycine → aspartate) | ~40% |
| G12V (glycine → valine) | ~30% |
| G12R (glycine → arginine) | ~15% |
| **G12C** (glycine → cysteine) | **~1–2%** |

Hold onto that last row. The first targeted KRAS drugs to reach the clinic only hit G12C — the *rarest* of the common variants in this disease. That mismatch is the whole story.

## RAS is a switch, and the cancer jams it on

RAS proteins (KRAS, NRAS, HRAS) are small **GTPases**: molecular switches that toggle between two states.

- **ON** — bound to GTP. Active. Recruits downstream effectors and tells the cell to grow.
- **OFF** — bound to GDP. Inactive. Quiet.

The cell flips the switch with helper proteins. **GEFs** (guanine-nucleotide exchange factors, like SOS) load fresh GTP and turn RAS *on*. **GAPs** (GTPase-activating proteins) accelerate RAS's own hydrolysis of GTP back to GDP and turn it *off*. In a healthy cell, RAS spends most of its time off, blipping on only when a growth signal arrives.

When RAS is on, it hands the signal down two main chains:

![Schematic of RAS signaling. Top: a two-state switch — a gray "RAS·GDP (OFF, inactive)" box on the left and a green, bold "RAS·GTP (ON, active)" box on the right, with a "GEF (SOS) loads GTP" arrow turning it on and a "GAP hydrolyzes GTP" arrow turning it off. A red mark notes that a G12/G13/Q61 mutation blocks the GAP, so RAS can't switch off and stays locked on. From the active RAS·GTP box, green arrows branch down into two effector cascades: a blue MAPK cascade (RAF → MEK → ERK → Proliferation) on the left, and a purple PI3K cascade (PI3K → AKT → mTOR → Survival and metabolism) on the right. A green callout in the center shows daraxonrasib plus cyclophilin A forming a tri-complex that clamps RAS·GTP, with a red blunt inhibition bar across the trunk showing it blocks the handoff to both cascades.](/blog/daraxonrasib/ras-signaling.png)

The RAF → MEK → ERK arm is the one most people mean when they say "the RAS pathway" — a relay that ends in the nucleus telling the cell to divide. The green callout at the center of the diagram previews where daraxonrasib breaks the chain; more on that two sections down.

The codon-12 mutations break the off switch. Position 12 sits in the **P-loop**, the part of the protein that cradles GTP's phosphates. Swap glycine for a bulkier residue and you sterically block the GAP from doing its job: RAS can no longer hydrolyze GTP efficiently, so it stays loaded, stays on, and the proliferation relay never stops firing. The cancer has jammed the switch in the on position.

## Forty years of "undruggable"

RAS was identified as a human oncogene in the early 1980s, and then resisted every attempt to drug it for roughly forty years. Two reasons:

1. **No pocket.** Good drug targets have a deep hydrophobic cleft a small molecule can wedge into. RAS is a relatively smooth globular protein. There was nowhere obvious to bind.
2. **You can't out-compete GTP.** The naive plan — block the nucleotide site — fails because RAS binds GTP with *picomolar* affinity while the cell is swimming in GTP at *millimolar* concentrations. A competitive inhibitor would need to be absurdly potent to win that fight.

So RAS earned a reputation as the canonical "undruggable" target.

The first crack came around 2013: the G12C mutation introduces a **cysteine**, a reactive handle absent from normal RAS, and a molecule can be built to bond covalently to it. That produced **sotorasib** and **adagrasib**, the first approved KRAS inhibitors — but with a catch that matters for pancreatic cancer. They grab RAS only in its **OFF (GDP-bound) state**, and only the **G12C** variant — the ~1–2% slice of PDAC. The G12D/G12V/G12R majority has no cysteine to target. For pancreatic cancer, the G12C era was a real milestone that was almost clinically irrelevant; you cannot move a disease by drugging 2% of it.

## RAS-ON: hitting the protein while it's working

Daraxonrasib comes at the problem from the opposite side. Instead of waiting for RAS to switch off, it targets the **active, GTP-bound, ON state directly** — the state that is doing the damage. Hence the name **RAS(ON) inhibitor**.

The mechanism is genuinely clever, and it sidesteps the "no pocket" problem by *building* one. Daraxonrasib doesn't bind RAS alone. It first binds an abundant intracellular chaperone protein, **cyclophilin A**. That drug–cyclophilin pair then clamps onto active RAS-GTP, forming a three-body **tri-complex**. The newly assembled surface physically gets in the way of RAS handing its signal to RAF and the other effectors. Active RAS is still on — but it has been muzzled. It's a molecular-glue strategy: the drug doesn't fit a natural cleft, it manufactures a binding interface that nature never left open.

The second, and for PDAC the decisive, property is in the word **multi-selective**. Daraxonrasib doesn't recognize one mutant allele. It clamps the shared active conformation across the **whole G12X family (G12D, G12V, G12R, …), plus G13 and Q61**, across KRAS, NRAS, and HRAS, and even wild-type RAS. One drug, nearly the entire pancreatic-cancer mutational landscape — the 90%, not the 2%.

That is the "big deal." Not "another KRAS drug," but a drug that (a) attacks the state of the protein that signals, and (b) is broad enough to be relevant to almost every patient with the disease. (Revolution Medicines also has allele-*selective* RAS(ON) inhibitors in development — RMC-6291 for G12C, RMC-9805 for G12D — but daraxonrasib is the pan-variant one.)

## The evidence

Two papers, building on each other.

### Phase 1/2 — the signal (*NEJM*, 2026)

The dose-finding study treated **168 patients** with previously treated RAS-mutated PDAC at 300 mg once daily. In a disease where second-line response rates are usually in the low single digits, the numbers were eye-catching:

- **Objective response rate:** 35% in the RAS **G12** subgroup (n=26); 29% across G12/G13/Q61 (n=38).
- In the G12/G13/Q61 group: **median overall survival 15.6 months**, median progression-free survival 8.1 months, median duration of response 8.2 months.
- Grade 3-or-higher treatment-related adverse events in 30% of patients.

A single-arm phase 1/2 can't prove a survival benefit — there's no control group, and early-phase populations are selected. But an ORR near 30–35% and a median OS over a year in *second-line* PDAC was, by the standards of this disease, a loud signal. It earned an FDA Breakthrough Therapy designation and justified a randomized trial.

### Phase 3 RASolute 302 — the proof ([*NEJM*, 2026](https://www.nejm.org/doi/full/10.1056/NEJMoa2605555))

This is the one that matters. **500 patients** with previously treated metastatic PDAC, randomized roughly 1:1 to once-daily oral daraxonrasib (n=248) or **investigator's choice of standard chemotherapy** (n=252, drawn from four regimens). A real control arm, a real comparison.

![Illustrative reconstruction of the RASolute 302 overall-survival curves: two survival curves descending from 100%, with daraxonrasib (deep green) staying consistently above chemotherapy (gray). Dashed guides mark the median overall survival where each curve crosses 50% — 6.7 months for chemotherapy and 13.2 months for daraxonrasib. A boxed annotation reads "HR 0.40 (P < .0001), 60% lower risk of death." A caption notes the curves are schematic exponential reconstructions anchored to the published medians, not the original Kaplan–Meier data.](/blog/daraxonrasib/rasolute-302-os.png)

The headline, in the intent-to-treat population:

- **Median overall survival: 13.2 months with daraxonrasib vs 6.7 months with chemotherapy.**
- **Hazard ratio 0.40 (P < .0001)** — a **60% reduction in the risk of death.**

It hit the same mark in the RAS G12 co-primary population (13.2 vs 6.6 months, HR 0.40). Progression-free survival roughly doubled as well — 7.3 vs 3.5 months in the G12 group (HR 0.45) and 7.2 vs 3.6 months overall (HR 0.49), both P < 0.001. Response rates tripled: about 32–33% with daraxonrasib vs ~11% with chemotherapy.

A hazard ratio of 0.40 is the part to sit with. In a disease where the win condition for a new drug has often been "a few extra weeks," a single agent cut the death rate by 60% and pushed median survival from under seven months to over thirteen.

> The curves above are a schematic I reconstructed from the published median OS and hazard ratio to show the *shape* of the separation — not the trial's actual Kaplan–Meier plot, which is copyrighted by the *New England Journal of Medicine* and not reproduced here. For the real, peer-reviewed survival figure, see **[Figure 1 of the *NEJM* paper](https://www.nejm.org/doi/full/10.1056/NEJMoa2605555)**.

### And it was gentler than chemo

Survival gains often come with a tolerability tax. Here it ran the other way. Grade 3-or-higher adverse events were **43.6% with daraxonrasib vs 57.5% with chemotherapy**, and discontinuation due to adverse events was **1.2% vs 11.2%**. The most characteristic side effect is rash — plausibly on-target, since the drug also touches wild-type RAS in normal tissue — but the overall profile was described as manageable and, notably, better than the comparator. A drug that both works better and is tolerated better is not the usual trade.

## What this doesn't settle

Measured enthusiasm, not a victory lap. A few honest caveats:

- **It is not a cure.** Median OS of 13.2 months is a large *relative* improvement in a dire setting, not long-term remission. The curve still bends down.
- **Durability and resistance are open.** Cancers route around targeted therapies; how and how fast RAS(ON)-inhibited tumors escape is still being mapped.
- **First-line and combinations are pending.** RASolute 302 is second-line monotherapy. Trials in the first-line setting and in combination (RASolute 303) will decide how much further this goes.
- **Regulatory status, as of this writing (June 2026), is pre-approval.** Breakthrough Therapy and Orphan Drug designations and expanded access exist; full approval does not yet. This is a fresh phase 3 readout, not yet a standard of care.

## Why I think the mechanism is the headline

It would be easy to file this under "good cancer-trial news" and move on. But the reason it works is more interesting than the fact that it works.

For forty years RAS was undruggable. The first era of progress (G12C, OFF-state, covalent) was real but narrow — it drugged the wrong state of the wrong 2% for pancreatic cancer. Daraxonrasib changes both variables at once: it confronts the **active state**, and it does so across **nearly the entire mutational spectrum** of the disease. The tri-complex / molecular-glue trick — manufacturing a binding surface with a borrowed chaperone instead of hunting for a natural pocket — is the kind of sideways solution that, in hindsight, looks obvious and, in foresight, took decades.

If RAS(ON) inhibition generalizes — to first-line PDAC, to RAS-driven lung and colorectal cancers, to combinations — then the forty-year "undruggable" label on the most common oncogene in human cancer is being quietly retired. That is the bet. RASolute 302 is the strongest evidence so far that it is the right one.

---

## Acknowledgment

My thanks to **Bogdan Popescu, MD, PhD** for his oncologic guidance on this piece, and for his own work on multi-selective RAS(ON) inhibition — notably [*Multiselective RAS(ON) inhibition targets oncogenic RAS and overcomes RAS-mediated resistance to FLT3i and BCL2i in AML*](https://ashpublications.org/blood/article/147/3/276/547938/) (*Blood*, 2026; [preprint](https://www.biorxiv.org/content/10.1101/2025.06.10.658786v1) posted June 2025). That work tests the **same RAS(ON) multi-selective approach** — using RMC-7977, the preclinical analog of daraxonrasib (RMC-6236) from the same Revolution Medicines program — in acute myeloid leukemia, and was carried out and published *before* the daraxonrasib pancreatic-cancer study described above. In other words, the strategy this post celebrates in PDAC was already being validated, in a different cancer, in his lab. Any errors that remain are mine.

## References

1. Wolpin BM, Park W, Garrido-Laguna I, et al. **Daraxonrasib in Previously Treated Advanced RAS-Mutated Pancreatic Cancer.** *N Engl J Med.* 2026;394(18):1790–1802. DOI: [10.1056/NEJMoa2505783](https://www.nejm.org/doi/full/10.1056/NEJMoa2505783)
2. **Daraxonrasib or Chemotherapy in Previously Treated Metastatic Pancreatic Cancer** (RASolute 302). *N Engl J Med.* 2026. DOI: [10.1056/NEJMoa2605555](https://www.nejm.org/doi/full/10.1056/NEJMoa2605555). Presented as Abstract LBA5, 2026 ASCO Annual Meeting plenary.
3. Revolution Medicines. [Daraxonrasib Demonstrates Unprecedented Overall Survival Benefit in Pivotal Phase 3 RASolute 302 Clinical Trial](https://ir.revmed.com/news-releases/news-release-details/daraxonrasib-demonstrates-unprecedented-overall-survival-benefit) (investor release).
4. ASCO. [Multi-Selective RAS(ON) Inhibitor Nearly Doubles Survival Time in People With Metastatic Pancreatic Cancer](https://www.asco.org/about-asco/press-center/multi-selective-ras-inhibitor-nearly-doubles-survival-pancreatic).
5. Popescu B, Jones MF, Piao M, et al. **Multiselective RAS(ON) inhibition targets oncogenic RAS and overcomes RAS-mediated resistance to FLT3i and BCL2i in AML.** *Blood.* 2026;147(3):276–289. [Article](https://ashpublications.org/blood/article/147/3/276/547938/) · [PubMed](https://pubmed.ncbi.nlm.nih.gov/40661530/)

*Background on RAS biology and the G12C OFF-state inhibitors drawn from the review literature on KRAS in pancreatic cancer; mutation-frequency figures are approximate and vary by series.*
