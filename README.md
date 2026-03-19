# Breach Risk Assessment Calculator

**Project:** In-house replacement for Radar  
**Owner:** Gawde & Gawde Computer Services  
**File:** `risk-calculator.html` — single-file, zero dependencies, deployable on GitHub Pages  
**Tree version:** v8  
**Verified test cases:** 43/43

---

## Overview

This calculator replicates the risk severity output of the Radar breach assessment tool. It takes 7 inputs about a data breach incident and returns a severity level of **LOW**, **MODERATE**, **HIGH**, or **EXTREME**.

The logic was reverse-engineered entirely from empirical testing — running known inputs through Radar, recording the outputs, and iterating until every case matched. The model is a **priority-ordered decision tree**, not a weighted score. It went through 8 iterations (v1–v8) across 6 test batches to reach 43/43 accuracy.

---

## The 7 Input Dimensions

| # | Dimension | Role in tree |
|---|---|---|
| 1 | Protection Measure | Gate 2 trigger — de-identified (score 0.10) changes the entire risk profile |
| 2 | Incident Nature | Filters Compromise dropdown; used in Gate 2 sub-rule for malicious intent |
| 3 | Compromise Description | Feeds the ransomware + hacker/media exception on Gates 1a and 1b |
| 4 | Recipient | Filters Recipient Description dropdown; determines authorized vs unauthorized |
| 5 | Recipient Description | Gate 1a legally-obligated check; Gate 2 hacker/media check; Gate 3 EXTREME vs HIGH |
| 6 | Disposition | Gate 3 and Gate 4 trigger — sufficient vs insufficient |
| 7 | Mitigation Name | **Most decisive input** — determines which gate fires and at what severity |

All three dependent dropdowns (Compromise, Recipient Description, Mitigation) cascade automatically based on their parent selection, mirroring exactly what Radar shows.

---

## Decision Tree Logic

Gates are evaluated top-to-bottom. The first gate that returns a non-null result wins — later gates are never evaluated.

```
Gate 1a → Gate 1b → Gate 2 → Gate 3 → Gate 4
```

---

### Gate 1a — Mitigation confirms data was not accessed

**Fires when** mitigation is one of: `unopened`, `forensic`, `backup`, `no_retain`  
**Also fires when** mitigation = `satisfactory` AND recipient is legally obligated (see LEGALLY_OBLIGATED set below)

**Normal result: LOW**

**Exception — ransomware + hacker/media + readable data → MODERATE**  
When `compromise = ransomware` AND `recipient = hacker or media` AND `protection > 0.10`, Gate 1a returns MODERATE instead of LOW. Ransomware can encrypt and exfiltrate silently — "unopened mail" does not apply because the malware does not require a human to open anything. This exception does not apply when protection is de-identified (0.10) because unreadable data neutralises the risk regardless.

**Rationale for the base rule:** These mitigations provide objective, verifiable proof that data never reached a harmful state. They are the only mitigations in the entire tree that give forensic or physical certainty. Radar treats them as a hard LOW overriding all other inputs — even plain text + malicious + hacker resolves LOW if data was forensically confirmed not accessed.

**Why `satisfactory` lives here (not Gate 1b):** The full mitigation label is "Satisfactory assurance was obtained from a recipient that is obligated to protect personal data by law and/or contract." This is only legally meaningful when the recipient actually has that obligation — covered entities, business associates, federal agencies. For recipients with no legal obligation (general public, vendor, attorney), `satisfactory` drops to Gate 1b and returns MODERATE.

---

### Gate 1b — Mitigation = assurance given (data may have been seen)

**Fires when** mitigation is one of: `permitted`, `written_obtained`, `written_provided`, `satisfactory` (non-obligated recipient), `obligated_no_assurance`, `ce_ba_obligated`, `no_written_obtained`, `no_written_provided`, `company_policy`

**Result:**
- Protection = de-identified (0.10) → **LOW**
- Protection = readable → **MODERATE**
- Exception: ransomware + hacker/media + readable → **MODERATE** (same exception as Gate 1a)

**Rationale:** These mitigations indicate data may have reached the recipient but containment was attempted through promises or legal obligation. Unlike Gate 1a, there is no proof of non-access — only assurance. Radar consistently returns MODERATE for readable data in this group.

**Why de-identified + assurance = LOW:** If data is de-identified, even a failed assurance causes no individual harm — the data cannot be traced back to a person. Radar applies this logic regardless of how bad the other inputs look.

**Key distinction:** Gate 1a = data provably never read. Gate 1b = data may have been read but was contained.

---

### Gate 2 — Protection = de-identified

**Fires when** `protection = 0.10` (Statistically de-identified) AND Gates 1a/1b did not fire

**Sub-rules (evaluated in order):**

| Condition | Result |
|---|---|
| Recipient = hacker/media AND nature = malicious (≥0.80) AND mitigation = `ransomware_confirmed` | HIGH |
| Recipient = hacker/media AND nature = malicious (≥0.80) AND mitigation in GATE2_BAD_MIT | MODERATE |
| Recipient = hacker/media AND nature = malicious (≥0.80) AND any other mitigation | LOW |
| Recipient = hacker/media AND nature NOT malicious (<0.80) | LOW |
| Recipient = anything else AND mitigation = `ransomware_confirmed` | MODERATE |
| Everything else | LOW |

**GATE2_BAD_MIT** = `unable_retrieve`, `improper_use`, `sent_to_media`, `unknown_mit`, `ssn_exposed`, `ransomware_confirmed`

**Rationale:** De-identified data cannot be traced back to an individual — default is LOW. Exceptions escalate only when a high-risk recipient received it with clear malicious intent. Incident nature matters here because an unintentional disclosure to media is fundamentally different from a deliberate theft by a hacker.

---

### Gate 3 — Readable data + insufficient disposition

**Fires when** `protection > 0.10` AND `disposition = Insufficient` AND Gates 1a/1b did not fire

**Sub-rules:**

| Condition | Result |
|---|---|
| Recipient type = unauthorized (any description) | EXTREME |
| Recipient type = authorized AND mitigation in G3_EXTREME | EXTREME |
| Recipient type = authorized AND mitigation in G3_HIGH | HIGH |
| Recipient type = authorized AND mitigation in G3_MODERATE | MODERATE |
| Recipient type = authorized AND any other mitigation | HIGH (default) |

**G3_EXTREME** = `improper_use`, `unknown_mit`, `sent_to_media`, `ssn_exposed`  
**G3_HIGH** = `unable_retrieve`  
**G3_MODERATE** = `confirmed_viewing`, `ransomware_confirmed`, `unsure_backup`

**Rationale — unauthorized always EXTREME:** Unauthorized recipients (vendors, hackers, media, general public, relatives, customers) have no legal accountability for the data. Readable data reaching them in insufficient disposition is always EXTREME.

**Rationale — authorized three-tier:** Authorized recipients (covered entities, business associates, federal agencies) have statutory obligations. Even with insufficient disposition, some constraint exists. The severity then depends on what we know happened:

- **EXTREME** = confirmed serious harm: data misused, catastrophic public exposure, or completely unknown outcome
- **HIGH** = unresolved: we could not retrieve the data and don't know what happened
- **MODERATE** = low evidence of harm: data was seen but no misuse confirmed, or uncertainty about copies

---

### Gate 4 — Readable data + sufficient disposition (fallthrough)

**Fires when** `protection > 0.10` AND `disposition = Sufficient` AND nothing fired above

**Result: MODERATE**

**Note:** In 43 verified test cases, Gate 4 has never fired in practice. Every sufficient mitigation in Radar belongs to either CONFIRMED_SAFE (Gate 1a) or ASSURED_SAFE (Gate 1b). Gate 4 is a safety net for any future mitigation option that does not fit either bucket.

---

## Mitigation Buckets — Complete Reference

### CONFIRMED_SAFE → Gate 1a → LOW
`unopened`, `forensic`, `backup`, `no_retain`  
Plus: `satisfactory` when recipient is in LEGALLY_OBLIGATED

### ASSURED_SAFE → Gate 1b → LOW (de-id) or MODERATE (readable)
`permitted`, `written_obtained`, `written_provided`, `satisfactory` (non-obligated recipient),  
`obligated_no_assurance`, `ce_ba_obligated`, `no_written_obtained`, `no_written_provided`, `company_policy`

### Gate 3 sub-buckets (authorized + insufficient only)

| Bucket | Keys | Result |
|---|---|---|
| G3_EXTREME | `improper_use`, `unknown_mit`, `sent_to_media`, `ssn_exposed` | EXTREME |
| G3_HIGH | `unable_retrieve` | HIGH |
| G3_MODERATE | `confirmed_viewing`, `ransomware_confirmed`, `unsure_backup` | MODERATE |

### LEGALLY_OBLIGATED recipients (affects Gate 1a satisfactory rule)
`covered_entity`, `business_associate`, `ba_state_gov`, `federal_agency`, `ohca`,  
`self_insured_sponsor`, `fully_insured_sponsor`, `reg_investment_advisor`, `state_gov`, `employee`

---

## Key Design Decisions

### Why a decision tree, not a weighted score

The first version used a weighted score across all 7 dimensions. It failed 3/3 on the first test batch. The core problem: Radar does not average inputs. Mitigation outcome dominates everything else. A case with the worst-possible protection, most malicious intent, and most dangerous recipient still returns LOW if forensic analysis confirmed no compromise. No weighted score can replicate that.

### Why mitigation is the most important input

Mitigation reflects what actually happened after the breach — the outcome, not the setup. Radar is fundamentally answering "given what we now know, how much harm occurred?" rather than "given the circumstances, how likely is harm?" Even terrible inputs (ransomware, plain text, hacker) can resolve LOW if the outcome was confirmed safe.

### Why `satisfactory` is split between Gate 1a and Gate 1b

The word "satisfactory" in the mitigation label is modified by "from a recipient that is obligated to protect personal data by law and/or contract." That legal obligation is what makes it reliable — for legally bound recipients it is equivalent to a physical confirmation, for others it is just a promise.

### Why the ransomware + hacker/media exception exists

Observed empirically from batch 6. Ransomware does not require a human to open a file — it encrypts and exfiltrates autonomously. The `unopened` mitigation was designed for physical mail and email contexts. For ransomware, the concept does not apply. Radar returns MODERATE in this specific combination rather than LOW.

### Why Gate 3 has three severity levels for authorized recipients

Discovered in batch 5. Previously all authorized + insufficient cases returned HIGH. Batches 4 and 5 revealed that the specific mitigation outcome within insufficient disposition changes the result — confirmed misuse is EXTREME, unresolved situation is HIGH, and low-evidence scenarios are MODERATE. The distinction reflects "what do we actually know happened?"

---

## Version History

| Version | Cases verified | Key change |
|---|---|---|
| v1 | 0/3 | Weighted score model — wrong approach |
| v2 | 3/3 | Decision tree introduced — mitigation as primary gate |
| v3 | 10/10 | Gate 2 refined — de-id sub-rules added |
| v4 | 19/19 | Gate 3 authorized vs unauthorized split; incident nature in Gate 2 |
| v5 | 28/28 | Gate 1 split into 1a (confirmed) and 1b (assured) |
| v6 | 34/34 | `satisfactory` splits by legal obligation; Gate 1b modulated by protection; Gate 2 mitigation sub-rules |
| v7 | 39/39 | Gate 3 authorized+insufficient splits into EXTREME / HIGH / MODERATE by mitigation |
| v8 | 43/43 | Ransomware + hacker/media exception on Gates 1a and 1b |

---

## Test Batches Summary

| Batch | Cases | Result | Key discovery |
|---|---|---|---|
| Original (O1–O3) | 3 | 0/3 → fixed | Weighted score wrong; tree approach needed |
| Batch 1 (P1–P7) | 7 | 7/7 | Gate 2 structure confirmed |
| Batch 2 (N1–N9) | 9 | 9/9 | Authorized vs unauthorized split in Gate 3 |
| Batch 3 (Q1–Q9) | 9 | 9/9 | Gate 1a/1b split discovered |
| Batch 4 (R1–R6) | 6 | 6/6 | `satisfactory` legal-obligation rule; de-id+assurance=LOW; Gate 2 mitigation refinement |
| Batch 5 (S1–S5) | 5 | 5/5 | Gate 3 three-tier split; N1 reading corrected |
| Batch 6 (T1–T4) | 4 | 4/4 | Ransomware+hacker/media exception |
| **Total** | **43** | **43/43** | |

---

## Cascading Dropdown Relationships

```
Incident Nature selected
    └─→ Compromise Description shows only valid options for that nature

Recipient selected
    └─→ Recipient Description shows only valid options for authorized or unauthorized

Disposition selected
    └─→ Mitigation Name shows only sufficient or insufficient mitigations
```

Mappings come directly from `Extracted_Mapping_Tables.xlsx` and match Radar exactly.

---

## Built-in Test Log

For each test case:

1. Select all 7 inputs to match what you entered in Radar
2. Click the LOW / MOD / HIGH / EXT button to record the Radar output
3. Click **+ Log Current Case** — saves with MATCH or MISMATCH and the gate that fired
4. **Export CSV** downloads the full log for offline analysis

---

## Multi-Country / Multi-Region Support

### What stays the same (zero code changes needed)

The tree only checks these structural signals — none reference label strings:

1. `protection ≤ 0.10` — de-identified
2. `disposition ≥ 0.70` — insufficient
3. `recip_type` = authorized or unauthorized
4. `mit_key` bucket membership — CONFIRMED_SAFE, ASSURED_SAFE, G3_EXTREME, G3_HIGH, G3_MODERATE
5. `nature ≥ 0.80` — malicious intent (Gate 2 sub-rule)
6. `rec_desc_key` in LEGALLY_OBLIGATED (Gate 1a satisfactory rule)
7. `compromise = ransomware` with hacker/media + readable (Gate 1a/1b exception)

### What changes (config layer)

Each region needs its own JSON config defining: protection scores, nature scores, compromise-by-nature mapping, recipient-by-type mapping, mitigation-by-disposition mapping, and the six bucket sets (CONFIRMED_SAFE, ASSURED_SAFE, G3_EXTREME, G3_HIGH, G3_MODERATE, LEGALLY_OBLIGATED).

### Estimated effort

| Task | Effort |
|---|---|
| Extract configs from HTML | 1–2 hours |
| Add region selector to UI | 2–3 hours |
| Map one new region (GDPR) | 3–5 hours config + empirical testing |
| Spring Boot config API | 1 day |
| Full multi-region production system | 3–5 days |

---

## File Structure

```
risk-calculator.html          ← single deployable file (GitHub Pages)
README.md                     ← this file
test_cases.json               ← all 43 verified cases with full inputs and RF results
new_test_cases.json           ← 72 generated test cases pending RF verification
remaining_test_cases.csv      ← remaining cases with v8 predictions, changed cases flagged
risk_test_log_*.csv           ← exported test logs from each testing batch
Extracted_Mapping_Tables.xlsx ← source of truth for dropdown cascade mappings
```

---

## How to Deploy on GitHub Pages

1. Rename `risk-calculator.html` to `index.html`
2. Push to a public GitHub repository
3. Go to **Settings → Pages → Source → main branch**
4. URL: `https://yourusername.github.io/repository-name/`

Fully self-contained — no backend, no build step, no npm install. Test log state is stored in `localStorage` and persists across sessions on the same device.

---

## Remaining Testing Priorities

The `remaining_test_cases.csv` file contains 72 cases still to verify. Run in this order:

1. **Prediction changed (17 cases marked YES)** — these changed between tree versions, most due to Gate 3 three-tier split and the ransomware exception. Highest priority.
2. **Gate 3 MODERATE cases** — `confirmed_viewing`, `ransomware_confirmed`, `unsure_backup` with authorized recipients, across multiple recipient descriptions.
3. **Gate 1a ransomware exception** — verify hacker/media exception applies with different nature scores and protection levels.
4. **Gate 2 mitigation refinement** — `unsure_backup` and `confirmed_viewing` with de-id + malicious + hacker/media should return LOW.
5. **Gate 4** — has never fired. If it fires, a new Radar mitigation option exists outside current buckets — investigate immediately.

---

## Spring Boot Integration Notes

The same 5-gate structure translates directly to Java:

```java
public String assessRisk(RiskInput input) {
    // Gate 1a — confirmed safe
    if (CONFIRMED_SAFE.contains(input.mitigationKey) ||
        ("satisfactory".equals(input.mitigationKey) && LEGALLY_OBLIGATED.contains(input.recipientDesc))) {
        return isRansomwareHighRisk(input) ? "MODERATE" : "LOW";
    }
    // Gate 1b — assured safe
    if (ASSURED_SAFE.contains(input.mitigationKey)) {
        String base = input.protectionScore <= 0.10 ? "LOW" : "MODERATE";
        return (isRansomwareHighRisk(input) && "LOW".equals(base)) ? "MODERATE" : base;
    }
    // Gate 2 — de-identified
    if (input.protectionScore <= 0.10) return assessGate2(input);
    // Gate 3 — readable + insufficient
    if (input.dispositionScore >= 0.70) return assessGate3(input);
    // Gate 4 — fallthrough
    return "MODERATE";
}
```

Bucket sets are Java `Set<String>` constants. No formulas, no weights — only set membership checks and two numeric threshold comparisons (`≤ 0.10` and `≥ 0.70`).
