# Breach Risk Assessment Calculator

**Project:** In-house replacement for Radar  
**Owner:** Gawde & Gawde Computer Services  
**File:** `index.html` — single-file, zero dependencies, deployable on GitHub Pages  
**Tree version:** v10  
**Verified test cases:** 48/48

---

## Overview

This calculator replicates the risk severity output of the Radar breach assessment tool. It takes 7 inputs about a data breach incident and returns a severity level of **LOW**, **MODERATE**, **HIGH**, or **EXTREME**.

The logic was reverse-engineered entirely from empirical testing — running known inputs through Radar, recording the outputs, and iterating until every case matched. The model is a **priority-ordered decision tree**, not a weighted score. It went through 10 iterations (v1–v10) across 7 test batches to reach 48/48 accuracy.

---

## The 7 Input Dimensions

| # | Dimension | Role in tree |
|---|---|---|
| 1 | Protection Measure | Gate 2 trigger — de-identified (score 0.10) changes the entire risk profile |
| 2 | Incident Nature | Filters Compromise dropdown; malicious intent (≥0.80) used in Gates 1a, 1b, and 2 |
| 3 | Compromise Description | No longer used as a direct gate trigger (ransomware-specific exception replaced by nature-based rule in v10) |
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

**Exception — malicious intent + hacker/media + readable data → MODERATE**  
When `nature ≥ 0.80` AND `recipient = hacker or media` AND `protection > 0.10`, Gate 1a returns MODERATE instead of LOW. This was originally documented as ransomware-specific (v8), but batch 7 (F5) confirmed it applies to any malicious compromise type — hacking, theft, and others. The underlying reason is the same: a malicious actor targeting a dangerous recipient cannot be contained by a "data not opened" assurance. This exception does not apply when protection is de-identified (0.10) because unreadable data neutralises the risk regardless.

**Rationale for the base rule:** These mitigations provide objective, verifiable proof that data never reached a harmful state. They are the only mitigations in the entire tree that give forensic or physical certainty. Radar treats them as a hard LOW overriding all other inputs — even plain text + malicious + hacker resolves LOW if data was forensically confirmed not accessed.

**Why `satisfactory` lives here (not Gate 1b):** The full mitigation label is "Satisfactory assurance was obtained from a recipient that is obligated to protect personal data by law and/or contract." This is only legally meaningful when the recipient actually has that obligation — covered entities, business associates, federal agencies — AND is an authorized recipient. For recipients with no legal obligation (general public, vendor, attorney), or for unauthorized recipients regardless of their description, `satisfactory` drops to Gate 1b.

**Why `no_written_obtained` and `no_written_provided` can also fire Gate 1a:** Batch 7 (F1, F2) revealed that when these mitigations are used with an authorized legally-obligated recipient, Radar returns LOW — not MODERATE. The rationale: "no written assurance obtained" in this context means no written form was collected *because the recipient's legal obligation already provides the guarantee*. It is semantically equivalent to `satisfactory` — the obligation substitutes for the paperwork. **Important distinction:** `obligated_no_assurance` and `ce_ba_obligated` do NOT get this treatment — Q6 (batch 3) confirmed they return MODERATE even for a covered entity. The difference is that `no_written_obtained/provided` implies some assurance existed but wasn't put in writing, whereas `obligated_no_assurance` means no assurance of any kind was obtained.

**Why the authorized check matters:** The `recip_type === 'authorized'` guard was added in v9. Several recipient descriptions (covered_entity, ba_state_gov, federal_agency, etc.) appear in both the authorized and unauthorized dropdowns. Without the guard, an unauthorized covered entity receiving data with `satisfactory` mitigation would incorrectly resolve LOW. The guard ensures legal obligation only benefits authorized recipients.

---

### Gate 1b — Mitigation = assurance given (data may have been seen)

**Fires when** mitigation is one of: `permitted`, `written_obtained`, `written_provided`, `satisfactory` (non-obligated or unauthorized recipient), `obligated_no_assurance`, `ce_ba_obligated`, `no_written_obtained` (non-obligated or unauthorized recipient), `no_written_provided` (non-obligated or unauthorized recipient), `company_policy`

**Result:**
- Protection = de-identified (0.10) → **LOW**
- Protection = readable AND recipient = hacker or media AND nature ≥ 0.80 → **EXTREME**
- Protection = readable AND recipient = hacker or media AND nature < 0.80 → **HIGH**
- Protection = readable AND all other recipients → **MODERATE**

**Rationale:** These mitigations indicate data may have reached the recipient but containment was attempted through promises or legal obligation. Unlike Gate 1a, there is no proof of non-access — only assurance.

**Why de-identified + assurance = LOW:** If data is de-identified, even a failed assurance causes no individual harm — the data cannot be traced back to a person. Radar applies this logic regardless of how bad the other inputs look.

**Why hacker/media recipients escalate beyond MODERATE (v10 change):** Batch 7 (F3, F4) showed that an assurance-level mitigation is effectively meaningless when the recipient is a hacker or media outlet — they have no legal obligation to honour it. Radar does not treat this as a generic MODERATE. Instead it escalates based on intent: if the incident was malicious (nature ≥ 0.80), the result is EXTREME; if it was not malicious, the result is HIGH. The intuition is that hacker/media recipients are structurally incapable of providing reliable containment, so the mitigation quality is irrelevant — only the threat level of the incident itself drives the severity.

**Key distinction:** Gate 1a = data provably never read. Gate 1b = data may have been read but containment was attempted.

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

**Note:** In 48 verified test cases, Gate 4 has never fired in practice. Every sufficient mitigation in Radar belongs to either Gate 1a or Gate 1b. Gate 4 is a safety net for any future mitigation option that does not fit either bucket. If it fires, investigate immediately — a new Radar option may have been added outside the current mapping.

---

## Mitigation Buckets — Complete Reference

### Gate 1a (CONFIRMED_SAFE) → LOW
`unopened`, `forensic`, `backup`, `no_retain`  
Plus: `satisfactory` when `recip_type = authorized` AND recipient is in LEGALLY_OBLIGATED  
Plus: `no_written_obtained`, `no_written_provided` when `recip_type = authorized` AND recipient is in LEGALLY_OBLIGATED

### Gate 1a exception → MODERATE (overrides LOW)
Any of the above when `nature ≥ 0.80` AND `rec_desc = hacker or media` AND `protection > 0.10`

### Gate 1b (ASSURED_SAFE) → LOW (de-id) or MODERATE/HIGH/EXTREME (readable)
`permitted`, `written_obtained`, `written_provided`, `satisfactory` (non-obligated or unauthorized recipient),  
`obligated_no_assurance`, `ce_ba_obligated`, `no_written_obtained` (non-obligated or unauthorized recipient),  
`no_written_provided` (non-obligated or unauthorized recipient), `company_policy`

Readable data result by recipient: hacker/media + malicious → EXTREME; hacker/media + not-malicious → HIGH; all others → MODERATE

### Gate 3 sub-buckets (authorized + insufficient only)

| Bucket | Keys | Result |
|---|---|---|
| G3_EXTREME | `improper_use`, `unknown_mit`, `sent_to_media`, `ssn_exposed` | EXTREME |
| G3_HIGH | `unable_retrieve` | HIGH |
| G3_MODERATE | `confirmed_viewing`, `ransomware_confirmed`, `unsure_backup` | MODERATE |

### LEGALLY_OBLIGATED recipients (affects Gate 1a satisfactory and no-written rules)
`covered_entity`, `business_associate`, `ba_state_gov`, `federal_agency`, `ohca`,  
`self_insured_sponsor`, `fully_insured_sponsor`, `reg_investment_advisor`, `state_gov`, `employee`

**Note:** These descriptions appear in both the authorized and unauthorized recipient dropdowns. LEGALLY_OBLIGATED membership only grants Gate 1a treatment when `recip_type = authorized`.

---

## Key Design Decisions

### Why a decision tree, not a weighted score

The first version used a weighted score across all 7 dimensions. It failed 3/3 on the first test batch. The core problem: Radar does not average inputs. Mitigation outcome dominates everything else. A case with the worst-possible protection, most malicious intent, and most dangerous recipient still returns LOW if forensic analysis confirmed no compromise. No weighted score can replicate that.

### Why mitigation is the most important input

Mitigation reflects what actually happened after the breach — the outcome, not the setup. Radar is fundamentally answering "given what we now know, how much harm occurred?" rather than "given the circumstances, how likely is harm?" Even terrible inputs (ransomware, plain text, hacker) can resolve LOW if the outcome was confirmed safe.

### Why `satisfactory` is split between Gate 1a and Gate 1b

The word "satisfactory" in the mitigation label is modified by "from a recipient that is obligated to protect personal data by law and/or contract." That legal obligation is what makes it reliable — for legally bound authorized recipients it is equivalent to a physical confirmation, for others it is just a promise.

### Why `no_written_obtained/provided` can resolve LOW for obligated authorized recipients but `obligated_no_assurance` cannot

Both describe situations where no written form was collected. The difference is subtle but empirically confirmed: `no_written_obtained` and `no_written_provided` imply that some form of assurance existed but was not reduced to writing — the legal obligation fills the gap. `obligated_no_assurance` means no assurance of any kind was obtained at all — the label explicitly says "no assurance was obtained." Q6 (covered entity + obligated_no_assurance) confirmed MODERATE. F1/F2 (federal agency and covered entity + no_written_obtained) confirmed LOW.

### Why the Gate 1a exception applies to all malicious intent, not just ransomware

Originally (v8) the exception was written specifically for `compromise = ransomware` after T2 showed ransomware + hacker + readable + unopened → MODERATE. Batch 7 (F5) showed hacking + hacker + readable + malicious + unopened also → MODERATE. The root cause is not the ransomware mechanism but the malicious intent: a bad actor deliberately targeting data cannot be assumed not to have accessed it regardless of what the mitigation says. `nature ≥ 0.80` is the correct trigger.

### Why Gate 1b escalates to EXTREME or HIGH for hacker/media recipients

An assurance-level mitigation (promise, policy, written agreement) requires the recipient to voluntarily comply. A hacker or media outlet has no legal obligation and no incentive to honor such an assurance. Radar therefore treats the mitigation as providing no real containment and evaluates severity based purely on the threat level — malicious intent yields EXTREME, non-malicious yields HIGH.

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
| v8 | 43/43 | Ransomware + hacker/media exception on Gate 1a |
| v9 | 43/43 | Fix: Gate 1a satisfactory guard requires `recip_type = authorized`; Gate 1b dead ransomware code removed |
| v10 | 48/48 | Fix: `no_written_obtained/provided` + authorized + obligated → LOW; Gate 1a exception broadened to `nature ≥ 0.80`; Gate 1b hacker/media escalation (HIGH/EXTREME by intent) |

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
| Batch 6 (T1–T4) | 4 | 4/4 | Ransomware+hacker/media exception on Gate 1a |
| Batch 7 (F1–F5) | 5 | 5/5 | no_written_obtained+obligated=LOW; Gate 1a exception broadened; Gate 1b hacker/media escalation |
| **Total** | **48** | **48/48** | |

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

**Regression suite (auto-runs on load):** All 48 verified cases run automatically against the current tree on page load. Results are shown in a table below the inputs. A green progress bar and 48/48 count confirm the tree is correct. If any case fails, it is highlighted in red with the expected vs actual result.

**Manual test log:** For new Radar cases you run empirically:
1. Select all 7 inputs to match what you entered in Radar
2. Click the LOW / MOD / HIGH / EXT button to record the Radar output
3. Click **+ Log Current Case** — saves with MATCH or MISMATCH and the gate that fired
4. **Export CSV** downloads the full log for offline analysis

When you find a mismatch, log it, export the CSV, and use it to diagnose which gate needs a new rule. The regression suite will catch any existing case that breaks when you update the tree.

---

## Multi-Country / Multi-Region Support

### What stays the same (zero code changes needed)

The tree only checks these structural signals — none reference label strings:

1. `protection ≤ 0.10` — de-identified
2. `disposition ≥ 0.70` — insufficient
3. `recip_type` = authorized or unauthorized
4. `mit_key` bucket membership — CONFIRMED_SAFE, ASSURED_SAFE, NO_WRITTEN_MITS, G3_EXTREME, G3_HIGH, G3_MODERATE
5. `nature ≥ 0.80` — malicious intent (Gates 1a, 1b, and 2)
6. `rec_desc_key` in LEGALLY_OBLIGATED (Gate 1a satisfactory and no-written rules)
7. `rec_desc_key` in {hacker, media} (Gate 1b escalation, Gate 2 sub-rules)

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
index.html                    ← single deployable file (GitHub Pages)
README.md                     ← this file
test_cases.json               ← all 48 verified cases with full inputs and RF results
new_test_cases.json           ← generated test cases pending RF verification
remaining_test_cases.csv      ← remaining cases with v10 predictions, changed cases flagged
risk_test_log_*.csv           ← exported test logs from each testing batch
Extracted_Mapping_Tables.xlsx ← source of truth for dropdown cascade mappings
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

Run new Radar cases in this order to improve coverage of known gaps:

1. **Gate 1b hacker/media + `nature = 0.45` (intentional, not malicious)** — F3 confirmed HIGH for unintentional. Verify intentional-not-malicious (0.45) also returns HIGH across different ASSURED_SAFE mitigations (written_obtained, company_policy, etc.).
2. **Gate 1b media recipient** — F3/F4 tested hacker. Run the same combinations with media as the recipient to confirm symmetry.
3. **Gate 1a broadened exception with media** — F5 tested hacking+hacker. Run hacking+media+malicious+CONFIRMED_SAFE to confirm MODERATE.
4. **Gate 1a `no_written_provided` + obligated** — F1/F2 verified `no_written_obtained`. Verify its sibling `no_written_provided` behaves identically for obligated authorized recipients.
5. **Gate 4** — has never fired. If it fires, a new Radar mitigation option exists outside current buckets — investigate immediately.

---

## Spring Boot Integration Notes

The same 5-gate structure translates directly to Java:

```java
public String assessRisk(RiskInput input) {
    String mit = input.mitigationKey;
    boolean isAuthorized = "authorized".equals(input.recipientType);
    boolean isLegallyObligated = LEGALLY_OBLIGATED.contains(input.recipientDesc);
    boolean isHackerMedia = Set.of("hacker","media").contains(input.recipientDesc);
    boolean isReadable = input.protectionScore > 0.10;
    boolean isMalicious = input.natureScore >= 0.80;

    // Gate 1a — confirmed safe
    boolean gate1aFires = CONFIRMED_SAFE.contains(mit)
        || ("satisfactory".equals(mit) && isAuthorized && isLegallyObligated)
        || (NO_WRITTEN_MITS.contains(mit) && isAuthorized && isLegallyObligated);

    if (gate1aFires) {
        // Broadened exception: any malicious intent + hacker/media + readable → MODERATE
        if (isMalicious && isHackerMedia && isReadable) return "MODERATE";
        return "LOW";
    }

    // Gate 1b — assured safe
    if (ASSURED_SAFE.contains(mit)) {
        if (!isReadable) return "LOW";  // de-identified
        if (isHackerMedia) return isMalicious ? "EXTREME" : "HIGH";
        return "MODERATE";
    }

    // Gate 2 — de-identified
    if (!isReadable) return assessGate2(input);

    // Gate 3 — readable + insufficient
    if (input.dispositionScore >= 0.70) return assessGate3(input);

    // Gate 4 — fallthrough
    return "MODERATE";
}
```

Bucket sets are Java `Set<String>` constants. No formulas, no weights — only set membership checks and two numeric threshold comparisons (`≤ 0.10` and `≥ 0.70`).
