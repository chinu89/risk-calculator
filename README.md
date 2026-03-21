# Breach Risk Assessment Calculator

**Project:** In-house replacement for Radar  
**Owner:** Gawde & Gawde Computer Services  
**File:** `index.html` — single-file, zero dependencies, deployable on GitHub Pages  
**Tree version:** v12  
**Verified test cases:** 54/54

---

## Overview

This calculator replicates the risk severity output of the Radar breach assessment tool. It takes 7 inputs about a data breach incident and returns a severity level of **LOW**, **MODERATE**, **HIGH**, or **EXTREME**.

The logic was reverse-engineered entirely from empirical testing — running known inputs through Radar, recording the outputs, and iterating until every case matched. The model is a **priority-ordered decision tree**, not a weighted score. It went through 12 iterations (v1–v12) across 9 test batches to reach 54/54 accuracy.

---

## The 7 Input Dimensions

| # | Dimension | Role in tree |
|---|---|---|
| 1 | Protection Measure | Gate 2 trigger — de-identified (score 0.10) changes the entire risk profile |
| 2 | Incident Nature | Filters Compromise dropdown; malicious intent (≥0.80) used in Gates 1a, 1b, and 2 |
| 3 | Compromise Description | No longer a direct gate trigger — ransomware-specific exception replaced by nature-based rule in v10 |
| 4 | Recipient | Filters Recipient Description dropdown; determines authorized vs unauthorized |
| 5 | Recipient Description | Gate 1a legally-obligated check; Gate 1b hacker/media escalation; Gate 2 hacker/media check; Gate 3 EXTREME vs HIGH |
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
**Also fires when** mitigation = `satisfactory` AND `recip_type = authorized` AND recipient is in LEGALLY_OBLIGATED  
**Also fires when** mitigation is `no_written_obtained` or `no_written_provided` AND `recip_type = authorized` AND recipient is in LEGALLY_OBLIGATED

**Normal result: LOW**

**Exception — malicious intent + unauthorized recipient + readable data → MODERATE**  
When `nature >= 0.80` AND `recip_type = unauthorized` AND `protection > 0.10`, Gate 1a returns MODERATE instead of LOW. This exception started as ransomware-specific (v8), was broadened to any malicious compromise type targeting hacker/media (v10, F5), then extended to cover all unauthorized recipients (v11, V2 — a non-hacker/media partner also triggered MODERATE). The root logic: a malicious actor cannot be assumed not to have accessed data regardless of what the mitigation says, and that applies to any unauthorized recipient. This exception does not apply when protection is de-identified (0.10) because unreadable data neutralises the risk regardless.

**Rationale for the base rule:** These mitigations provide objective, verifiable proof that data never reached a harmful state. They are the only mitigations in the entire tree that give forensic or physical certainty. Radar treats them as a hard LOW overriding all other inputs — even plain text + malicious + hacker resolves LOW if data was forensically confirmed not accessed.

**Why `satisfactory` lives here (not Gate 1b):** The full mitigation label is "Satisfactory assurance was obtained from a recipient that is obligated to protect personal data by law and/or contract." This is only legally meaningful when the recipient actually has that obligation — covered entities, business associates, federal agencies — AND is an authorized recipient. For recipients with no legal obligation, or for unauthorized recipients regardless of their description, `satisfactory` drops to Gate 1b.

**Why `no_written_obtained` and `no_written_provided` can also fire Gate 1a:** Batch 7 (F1, F2) revealed that when these mitigations are used with an authorized legally-obligated recipient, Radar returns LOW. The rationale: "no written assurance obtained" in this context means no written form was collected because the recipient's legal obligation already provides the guarantee. It is semantically equivalent to `satisfactory`. **Important distinction:** `obligated_no_assurance` and `ce_ba_obligated` do NOT get this treatment — Q6 (batch 3) confirmed they return MODERATE even for a covered entity. The difference: `no_written_obtained/provided` implies some assurance existed but wasn't put in writing, whereas `obligated_no_assurance` explicitly says no assurance of any kind was obtained.

**Why the authorized check matters:** Several recipient descriptions (covered_entity, ba_state_gov, federal_agency, etc.) appear in both the authorized and unauthorized dropdowns. Without the guard, an unauthorized covered entity with `satisfactory` mitigation would incorrectly resolve LOW. Legal obligation only benefits authorized recipients.

---

### Gate 1b — Mitigation = assurance given (data may have been seen)

**Fires when** mitigation is one of: `permitted`, `written_obtained`, `written_provided`, `satisfactory` (non-obligated or unauthorized recipient), `obligated_no_assurance`, `ce_ba_obligated`, `no_written_obtained` (non-obligated or unauthorized recipient), `no_written_provided` (non-obligated or unauthorized recipient), `company_policy`

**Result:**
- Protection = de-identified (0.10) → **LOW**
- Protection = readable AND recipient = hacker or media AND nature >= 0.80 → **EXTREME**
- Protection = readable AND recipient = hacker or media AND nature < 0.80 → **HIGH**
- Protection = readable AND any other unauthorized recipient AND nature >= 0.80 → **HIGH**
- Protection = readable AND authorized recipient → **MODERATE**

**Rationale:** These mitigations indicate data may have reached the recipient but containment was attempted through promises or legal obligation. Unlike Gate 1a, there is no proof of non-access — only assurance.

**Why de-identified + assurance = LOW:** If data is de-identified, even a failed assurance causes no individual harm — the data cannot be traced back to a person.

**Why hacker/media recipients escalate to EXTREME or HIGH:** An assurance-level mitigation is meaningless when the recipient is a hacker or media outlet — they have no legal obligation to honour it. Radar escalates based on intent: malicious (>=0.80) → EXTREME; not malicious → HIGH. Discovered batch 7 (F3, F4).

**Why other unauthorized recipients also escalate to HIGH when malicious:** Batch 8 (V1, V3) showed that non-hacker/media unauthorized recipients — a partner and an unauthorized business associate — also return HIGH when intent is malicious and data is readable. Any unauthorized recipient receiving readable data via an assurance-only mitigation with malicious intent cannot reliably contain the data. Hacker/media remain the EXTREME sub-case; all other unauthorized malicious cases are HIGH.

**Key distinction:** Gate 1a = data provably never read. Gate 1b = data may have been read but containment was attempted.

---

### Gate 2 — Protection = de-identified

**Fires when** `protection = 0.10` (Statistically de-identified) AND Gates 1a/1b did not fire

**Sub-rules (evaluated in order):**

| Condition | Result |
|---|---|
| Recipient = hacker/media AND nature = malicious (>=0.80) AND mitigation = `ransomware_confirmed` | HIGH |
| Recipient = hacker/media AND nature = malicious (>=0.80) AND mitigation in GATE2_BAD_MIT | MODERATE |
| Recipient = hacker/media AND nature = malicious (>=0.80) AND any other mitigation | LOW |
| Recipient = hacker/media AND nature NOT malicious (<0.80) | LOW |
| Recipient = anything else AND mitigation = `ransomware_confirmed` | MODERATE |
| Everything else | LOW |

**GATE2_BAD_MIT** = `unable_retrieve`, `improper_use`, `sent_to_media`, `unknown_mit`, `ssn_exposed`, `ransomware_confirmed`

**Rationale:** De-identified data cannot be traced back to an individual — default is LOW. Exceptions escalate only when a high-risk recipient received it with clear malicious intent.

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
**G3_HIGH** = `unable_retrieve`, `confirmed_viewing`, `ransomware_confirmed`  
**G3_MODERATE** = `unsure_backup`

**Rationale — unauthorized always EXTREME:** Unauthorized recipients have no legal accountability. Readable data reaching them in insufficient disposition is always EXTREME.

**Rationale — authorized three-tier:** Authorized recipients have statutory obligations. The severity depends on what we know happened:

- **EXTREME** = confirmed serious harm: data misused, catastrophic public exposure, or completely unknown outcome
- **HIGH** = data was accessed or confirmed acquired, but no misuse yet confirmed — `confirmed_viewing`, `ransomware_confirmed`, `unable_retrieve`
- **MODERATE** = genuine uncertainty: destroyed but backup/copy status unknown (`unsure_backup`) — not confirmed accessed

**Why `confirmed_viewing` and `ransomware_confirmed` are HIGH, not MODERATE (v12 correction):** Originally placed in G3_MODERATE in batch 5, re-testing in batch 9 (C1, C2, C3) showed all cases return HIGH across different recipient descriptions and nature scores. The correction makes semantic sense: `confirmed_viewing` means access was confirmed (data was seen, misuse not yet detected), and `ransomware_confirmed` means data integrity is definitively at risk. Both represent a known bad outcome, not mere uncertainty. Only `unsure_backup` — where we genuinely don't know if the data was accessed — retains the MODERATE classification.

---

### Gate 4 — Readable data + sufficient disposition (fallthrough)

**Fires when** `protection > 0.10` AND `disposition = Sufficient` AND nothing fired above

**Result: MODERATE**

**Note:** In 54 verified test cases, Gate 4 has never fired in practice. Every sufficient mitigation belongs to either Gate 1a or Gate 1b. Gate 4 is a safety net — if it fires, a new Radar mitigation option exists outside current buckets and warrants immediate investigation.

---

## Mitigation Buckets — Complete Reference

### Gate 1a (CONFIRMED_SAFE) → LOW
`unopened`, `forensic`, `backup`, `no_retain`  
Plus: `satisfactory` when `recip_type = authorized` AND recipient is in LEGALLY_OBLIGATED  
Plus: `no_written_obtained`, `no_written_provided` when `recip_type = authorized` AND recipient is in LEGALLY_OBLIGATED

### Gate 1a exception → MODERATE (overrides LOW)
Any of the above when `nature >= 0.80` AND `recip_type = unauthorized` AND `protection > 0.10`

### Gate 1b (ASSURED_SAFE) → varies by recipient and protection
`permitted`, `written_obtained`, `written_provided`, `satisfactory` (non-obligated or unauthorized recipient),  
`obligated_no_assurance`, `ce_ba_obligated`, `no_written_obtained` (non-obligated or unauthorized recipient),  
`no_written_provided` (non-obligated or unauthorized recipient), `company_policy`

Readable data result: hacker/media + malicious → EXTREME; hacker/media + not-malicious → HIGH; other unauthorized + malicious → HIGH; authorized → MODERATE; de-identified (any) → LOW

### Gate 3 sub-buckets (authorized + insufficient only)

| Bucket | Keys | Result |
|---|---|---|
| G3_EXTREME | `improper_use`, `unknown_mit`, `sent_to_media`, `ssn_exposed` | EXTREME |
| G3_HIGH | `unable_retrieve`, `confirmed_viewing`, `ransomware_confirmed` | HIGH |
| G3_MODERATE | `unsure_backup` | MODERATE |

### LEGALLY_OBLIGATED recipients (affects Gate 1a satisfactory and no-written rules)
`covered_entity`, `business_associate`, `ba_state_gov`, `federal_agency`, `ohca`,  
`self_insured_sponsor`, `fully_insured_sponsor`, `reg_investment_advisor`, `state_gov`, `employee`

**Note:** These appear in both the authorized and unauthorized dropdowns. LEGALLY_OBLIGATED membership only grants Gate 1a treatment when `recip_type = authorized`.

---

## Key Design Decisions

### Why a decision tree, not a weighted score

The first version used a weighted score across all 7 dimensions. It failed 3/3 on the first test batch. Radar does not average inputs — mitigation outcome dominates everything else. A case with the worst-possible protection, most malicious intent, and most dangerous recipient still returns LOW if forensic analysis confirmed no compromise.

### Why mitigation is the most important input

Mitigation reflects what actually happened after the breach — the outcome, not the setup. Radar is fundamentally answering "given what we now know, how much harm occurred?" rather than "how likely is harm given the circumstances?"

### Why `satisfactory` is split between Gate 1a and Gate 1b

The mitigation label is "Satisfactory assurance from a recipient obligated by law and/or contract." That obligation only makes it reliable for legally bound authorized recipients — for others it is just a promise.

### Why `no_written_obtained/provided` resolves LOW for obligated authorized recipients but `obligated_no_assurance` does not

`no_written_obtained/provided` implies assurance existed but wasn't put in writing — the legal obligation fills the gap. `obligated_no_assurance` explicitly says no assurance of any kind was obtained. Q6 (covered entity + obligated_no_assurance) confirmed MODERATE. F1/F2 (federal agency and covered entity + no_written_obtained) confirmed LOW.

### Why the Gate 1a exception applies to all unauthorized recipients, not just hacker/media

Started as ransomware-specific (v8), broadened to any malicious intent + hacker/media (v10), then to all unauthorized (v11). V2 confirmed a partner also triggers MODERATE with malicious intent and readable data. The trigger is malicious intent + unauthorized status, not the specific recipient description.

### Why Gate 1b escalates for unauthorized recipients when intent is malicious

An unauthorized recipient has no structural incentive to honor an assurance, especially when the incident was intentional. Hacker/media are the extreme sub-case (EXTREME/HIGH by intent), but any other unauthorized + malicious scenario also escalates to HIGH. Authorized recipients stay at MODERATE because statutory obligations provide meaningful containment.

### Why Gate 3 G3_MODERATE now contains only `unsure_backup`

`confirmed_viewing` and `ransomware_confirmed` were moved from G3_MODERATE to G3_HIGH in v12 after batch 9 re-testing confirmed all original batch 5 cases (S1, S4, S5) return HIGH. `confirmed_viewing` means access was confirmed; `ransomware_confirmed` means data integrity is at risk. Both are known bad outcomes. Only `unsure_backup` — genuine uncertainty about whether data was accessed at all — retains MODERATE.

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
| v8 | 43/43 | Ransomware + hacker/media exception on Gate 1a |
| v9 | 43/43 | Fix: Gate 1a satisfactory guard requires `recip_type = authorized`; Gate 1b dead ransomware code removed |
| v10 | 48/48 | Fix: `no_written_obtained/provided` + authorized + obligated → LOW; Gate 1a exception to `nature >= 0.80` + hacker/media; Gate 1b hacker/media escalation |
| v11 | 51/51 | Gate 1a exception broadened to all unauthorized; Gate 1b: any unauthorized + malicious → HIGH |
| v12 | 54/54 | Correction: `confirmed_viewing` and `ransomware_confirmed` moved G3_MODERATE → G3_HIGH; S1/S4/S5 corrected; `unsure_backup` is now the only G3_MODERATE key |

---

## Test Batches Summary

| Batch | Cases | Result | Key discovery |
|---|---|---|---|
| Original (O1–O3) | 3 | 0/3 → fixed | Weighted score wrong; tree approach needed |
| Batch 1 (P1–P7) | 7 | 7/7 | Gate 2 structure confirmed |
| Batch 2 (N1–N9) | 9 | 9/9 | Authorized vs unauthorized split in Gate 3 |
| Batch 3 (Q1–Q9) | 9 | 9/9 | Gate 1a/1b split discovered |
| Batch 4 (R1–R6) | 6 | 6/6 | `satisfactory` legal-obligation rule; de-id+assurance=LOW; Gate 2 mitigation refinement |
| Batch 5 (S1–S5) | 5 | 5/5 | Gate 3 three-tier split — **S1/S4/S5 corrected in batch 9 (MODERATE→HIGH)** |
| Batch 6 (T1–T4) | 4 | 4/4 | Ransomware+hacker/media exception on Gate 1a |
| Batch 7 (F1–F5) | 5 | 5/5 | `no_written_obtained`+obligated+authorized=LOW; Gate 1a exception broadened; Gate 1b hacker/media escalation |
| Batch 8 (V1–V3) | 3 | 3/3 | Gate 1b: other unauthorized+malicious=HIGH; Gate 1a exception extended to all unauthorized |
| Batch 9 (C1–C3 + S1/S4/S5 corrections) | 3 new + 3 re-verified | 6/6 | `confirmed_viewing` and `ransomware_confirmed` corrected to G3_HIGH; `unsure_backup` only remaining G3_MODERATE |
| **Total** | **54** | **54/54** | |

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

## Built-in Test Log and Regression Runner

The calculator includes two test facilities:

**Regression suite (auto-runs on load):** All 54 verified cases run automatically against the current tree on page load. A green progress bar and 54/54 count confirm the tree is correct. If any case fails, it is highlighted in red with expected vs actual result shown.

**Manual test log:** For new Radar cases you run empirically:
1. Select all 7 inputs to match what you entered in Radar
2. Click the LOW / MOD / HIGH / EXT button to record the Radar output
3. Click **+ Log Current Case** — saves with MATCH or MISMATCH and the gate that fired
4. **Export CSV** downloads the full log for offline analysis

When you find a mismatch, log it, export the CSV, and use it to diagnose which gate needs a rule change or correction. The regression suite will immediately catch any existing case that breaks when you update the tree.

---

## Excel Test Case Workbook

`breach_risk_test_cases.xlsx` contains all 54 verified cases across three sheets:

- **All Verified Cases** — every case with full inputs, v12 prediction, Radar result, and match status. Rows colour-coded by batch. Corrected cases (S1, S4, S5) flagged with amber status.
- **Radar Re-test Checklist** — same cases with column L pre-filled with known Radar results and editable. Column M auto-calculates MATCH/MISMATCH as a formula. Amber-tinted rows indicate corrected cases to prioritise for re-verification.
- **Batch Summary** — batch-by-batch discovery log, v12 G3 bucket change callout, severity distribution, and gate distribution across all 54 cases.

---

## Multi-Country / Multi-Region Support

### What stays the same (zero code changes needed)

The tree only checks these structural signals — none reference label strings:

1. `protection <= 0.10` — de-identified
2. `disposition >= 0.70` — insufficient
3. `recip_type` = authorized or unauthorized
4. `mit_key` bucket membership — CONFIRMED_SAFE, ASSURED_SAFE, NO_WRITTEN_MITS, G3_EXTREME, G3_HIGH, G3_MODERATE
5. `nature >= 0.80` — malicious intent (Gates 1a, 1b, and 2)
6. `rec_desc_key` in LEGALLY_OBLIGATED (Gate 1a satisfactory and no-written rules)
7. `rec_desc_key` in {hacker, media} (Gate 1b hacker/media escalation, Gate 2 sub-rules)

### What changes (config layer)

Each region needs its own JSON config defining: protection scores, nature scores, compromise-by-nature mapping, recipient-by-type mapping, mitigation-by-disposition mapping, and the bucket sets (CONFIRMED_SAFE, ASSURED_SAFE, NO_WRITTEN_MITS, G3_EXTREME, G3_HIGH, G3_MODERATE, LEGALLY_OBLIGATED, and the hacker/media equivalents for that jurisdiction).

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
index.html                        <- single deployable file (GitHub Pages)
README.md                         <- this file
breach_risk_test_cases.xlsx       <- all 54 verified cases, re-test checklist, batch summary
risk_test_log_*.csv               <- exported test logs from each testing batch
Extracted_Mapping_Tables.xlsx     <- source of truth for dropdown cascade mappings
```

---

## How to Deploy on GitHub Pages

1. Ensure the file is named `index.html`
2. Push to a public GitHub repository
3. Go to **Settings → Pages → Source → main branch**
4. URL: `https://yourusername.github.io/repository-name/`

Fully self-contained — no backend, no build step, no npm install. Manual test log state is stored in `localStorage` and persists across sessions on the same device.

---

## Remaining Testing Priorities

1. **Gate 3 `unsure_backup` confirmation** — now the only G3_MODERATE key. Verify it still returns MODERATE across multiple authorized recipient descriptions (CE, BA, federal agency) to confirm it was not also affected by the batch 9 correction.
2. **Gate 1b media + malicious** — V1/V3 confirmed partner and unauthorized business_associate return HIGH. Run `media` as recipient + malicious intent + ASSURED_SAFE to confirm EXTREME (not just HIGH).
3. **Gate 1b unauthorized + not malicious (nature = 0.45)** — verify intentional-not-malicious + unauthorized + readable + ASSURED_SAFE returns MODERATE (not HIGH), confirming malicious intent is the correct dividing line.
4. **Gate 1a `no_written_provided` + obligated authorized** — F1/F2 verified `no_written_obtained`. Verify its sibling behaves identically.
5. **Gate 4** — has never fired. If it fires, a new Radar mitigation option exists outside current buckets — investigate immediately.

---

## Spring Boot Integration Notes

```java
public String assessRisk(RiskInput input) {
    String mit = input.mitigationKey;
    boolean isAuthorized   = "authorized".equals(input.recipientType);
    boolean isUnauthorized = "unauthorized".equals(input.recipientType);
    boolean isLegallyObligated = LEGALLY_OBLIGATED.contains(input.recipientDesc);
    boolean isHackerMedia  = Set.of("hacker","media").contains(input.recipientDesc);
    boolean isReadable     = input.protectionScore > 0.10;
    boolean isMalicious    = input.natureScore >= 0.80;

    // Gate 1a — confirmed safe
    boolean gate1aFires = CONFIRMED_SAFE.contains(mit)
        || ("satisfactory".equals(mit) && isAuthorized && isLegallyObligated)
        || (NO_WRITTEN_MITS.contains(mit) && isAuthorized && isLegallyObligated);

    if (gate1aFires) {
        // Exception: malicious + unauthorized + readable -> MODERATE
        if (isMalicious && isUnauthorized && isReadable) return "MODERATE";
        return "LOW";
    }

    // Gate 1b — assured safe
    if (ASSURED_SAFE.contains(mit)) {
        if (!isReadable) return "LOW";
        if (isHackerMedia)               return isMalicious ? "EXTREME" : "HIGH";
        if (isUnauthorized && isMalicious) return "HIGH";
        return "MODERATE";
    }

    // Gate 2 — de-identified
    if (!isReadable) return assessGate2(input);

    // Gate 3 — readable + insufficient
    if (input.dispositionScore >= 0.70) return assessGate3(input);

    // Gate 4 — fallthrough
    return "MODERATE";
}

private String assessGate3(RiskInput input) {
    if ("unauthorized".equals(input.recipientType)) return "EXTREME";
    if (G3_EXTREME.contains(input.mitigationKey))   return "EXTREME";
    if (G3_HIGH.contains(input.mitigationKey))      return "HIGH";    // confirmed_viewing, ransomware_confirmed, unable_retrieve
    if (G3_MODERATE.contains(input.mitigationKey))  return "MODERATE"; // unsure_backup only
    return "HIGH"; // default for authorized + insufficient + unmapped mitigation
}

// v12 bucket sets
static final Set<String> G3_HIGH     = Set.of("unable_retrieve","confirmed_viewing","ransomware_confirmed");
static final Set<String> G3_MODERATE = Set.of("unsure_backup");
```

No formulas, no weights — only set membership checks and two numeric threshold comparisons (`<= 0.10` and `>= 0.70`).
