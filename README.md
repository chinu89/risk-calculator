# Breach Risk Assessment Calculator

**Project:** In-house replacement for Radar  
**Owner:** Gawde & Gawde Computer Services  
**File:** `index.html` — single-file, zero dependencies, deployable on GitHub Pages  
**Tree version:** v13  
**Verified test cases:** 82/82

---

## Overview

This calculator replicates the risk severity output of the Radar breach assessment tool. It takes 8 inputs about a data breach incident and returns a severity level of **LOW**, **MODERATE**, **HIGH**, or **EXTREME**.

The logic was reverse-engineered entirely from empirical testing — running known inputs through Radar, recording the outputs, and iterating until every case matched. The model is a **priority-ordered decision tree**, not a weighted score. It went through 13 iterations (v1–v13) across 10 test batches to reach 82/82 accuracy.

---

## The 8 Input Dimensions

| # | Dimension | Role in tree |
|---|---|---|
| 1 | Category | Filters Protection Measure dropdown (Paper or Electronic) |
| 2 | Protection Measure | Gate 2 trigger (de-id ≤ 0.10); Gate 1b authorized result; Gate 1a exception threshold |
| 3 | Incident Nature | Malicious (≥0.80) used in Gates 1a, 1b, 2, and 3 |
| 4 | Compromise Description | No longer a direct gate trigger — ransomware-specific exception replaced by nature-based rule in v10 |
| 5 | Recipient | Determines authorized vs unauthorized |
| 6 | Recipient Description | Gate 1a legally-obligated check; Gate 1b escalation tier; Gate 3 sub-rules |
| 7 | Disposition | Gate 3 and Gate 4 trigger — sufficient vs insufficient |
| 8 | Mitigation Name | **Most decisive input** — determines which gate fires and at what severity |

---

## Decision Tree Logic

```
Gate 1a → Gate 1b → Gate 2 → Gate 3 → Gate 4
```

Gates evaluated top-to-bottom. First non-null result wins.

---

### Gate 1a — Mitigation confirms data was not accessed

**Fires when:**
- mitigation in CONFIRMED_SAFE: `unopened`, `forensic`, `backup`, `no_retain`
- OR `satisfactory` AND `recip_type = authorized` AND rec_desc in LEGALLY_OBLIGATED
- OR `no_written_obtained` AND `recip_type = authorized` AND rec_desc in {`covered_entity`, `federal_agency`} AND `nature < 0.45`

**Normal result: LOW**

**Exception — unauthorized + readable: result depends on specific mitigation**

| Mitigation | Unauthorized + malicious | Unauthorized + not-malicious | Unauthorized + unintentional |
|---|---|---|---|
| `forensic` | LOW | LOW | LOW |
| `unopened` | MODERATE (hacker/media only); LOW (others) | LOW | LOW |
| `no_retain` | MODERATE | LOW | LOW |
| `backup` | **HIGH** | MODERATE | LOW |

**Why each mitigation behaves differently:** `forensic` provides algorithmic certainty — data provably not accessed regardless of who received it. `backup` is weaker: data existed and was recoverable, so a malicious actor could have exfiltrated it even if the specific copy was restored. `no_retain` falls between these two. `unopened` is physical mail semantics — a hacker doesn't need to "open" anything; other unauthorized recipients are more like physical mail scenarios.

**Why `no_written_obtained` → LOW only for CE/federal_agency, non-malicious:** Batch 10 (PT020) showed OHCA + no_written_obtained → MODERATE despite OHCA being in LEGALLY_OBLIGATED. The rule is specific to covered entities and federal agencies with non-malicious intent, not the full obligated set.

**Why `no_written_provided` was removed from Gate 1a entirely:** All tested cases with `no_written_provided` returned MODERATE for authorized recipients regardless of protection level. It describes data being sent outbound with no written form — weaker than `no_written_obtained`.

---

### Gate 1b — Mitigation = assurance given (data may have been seen)

**Fires when** mitigation in ASSURED_SAFE: `permitted`, `written_obtained`, `written_provided`, `satisfactory` (non-obligated or unauthorized), `obligated_no_assurance`, `ce_ba_obligated`, `no_written_obtained` (non-obligated or unauthorized), `no_written_provided` (non-obligated or unauthorized), `company_policy`

**Results by recipient type and protection level:**

**De-identified (≤ 0.10):** LOW always.

**Employee** (listed as unauthorized but treated as authorized-like): malicious → MODERATE; unintentional/not-malicious → LOW.

**Authorized recipients:**
- Protection = 0.85 (plain text) AND mitigation NOT in AUTH_WEAK_MITS → **LOW**
- Protection = 0.85 AND mitigation in AUTH_WEAK_MITS → **MODERATE**
- Protection < 0.85 (any other readable level) → **MODERATE** regardless of mitigation

**AUTH_WEAK_MITS** = `obligated_no_assurance`, `no_written_obtained`, `no_written_provided`

**Attorney** (unauthorized but extreme-tier): malicious → EXTREME; else → HIGH.

**Hacker or media** (protection + intent dependent):

| Nature | Protection ≤ 0.20 | Protection > 0.20 |
|---|---|---|
| Malicious (≥ 0.80) | EXTREME | HIGH |
| Unintentional (≤ 0.30) | HIGH | HIGH |
| Intentional, not malicious (0.45) | HIGH | MODERATE |

**Other unauthorized:** malicious → HIGH; else → MODERATE.

**Why plain text + authorized + strong assurance = LOW:** When data is already in plain text and reaches an authorized entity who provides a written assurance or confirmed permitted use, the risk is actually lower than when partially-protected data reaches the same recipient — the authorized entity's statutory obligation is sufficient containment. Q6/Q7/Q9 (protection 0.20–0.25) confirmed MODERATE; PT038–PT044 (protection 0.85) confirmed LOW.

**Why protection level determines hacker/media escalation:** F4 (redacted/0.20 + hacker + malicious) → EXTREME; PT021/PT022 (plain text/0.85 + hacker + malicious) → HIGH. The more readable the data, paradoxically, the lower the Gate 1b escalation tier — because higher protection scores reflect a worse breach while lower protection scores paradoxically indicate stronger safeguards.

---

### Gate 2 — Protection = de-identified

**Fires when** `protection ≤ 0.10` AND Gates 1a/1b did not fire.

| Condition | Result |
|---|---|
| hacker/media AND malicious AND `ransomware_confirmed` | HIGH |
| hacker/media AND malicious AND mitigation in GATE2_BAD_MIT | MODERATE |
| hacker/media AND malicious AND any other mitigation | LOW |
| hacker/media AND not malicious | LOW |
| anything else AND `ransomware_confirmed` | MODERATE |
| Everything else | LOW |

**GATE2_BAD_MIT** = `unable_retrieve`, `improper_use`, `sent_to_media`, `unknown_mit`, `ssn_exposed`, `ransomware_confirmed`

---

### Gate 3 — Readable data + insufficient disposition

**Fires when** `protection > 0.10` AND `disposition ≥ 0.70` AND Gates 1a/1b did not fire.

**Unauthorized recipients:**

| Nature | Recipient description | Mitigation | Result |
|---|---|---|---|
| Malicious or not-malicious | Any | Any | EXTREME |
| Unintentional (≤ 0.30) | In LEGALLY_OBLIGATED | Any | HIGH |
| Unintentional (≤ 0.30) | NOT in LEGALLY_OBLIGATED | G3_EXTREME | EXTREME |
| Unintentional (≤ 0.30) | NOT in LEGALLY_OBLIGATED | G3_HIGH or other | HIGH |

**Authorized recipients:**

| Condition | Result |
|---|---|
| Mitigation in G3_EXTREME | EXTREME |
| `ransomware_confirmed` AND nature ≥ 0.80 | EXTREME |
| `confirmed_viewing` AND rec_desc in LEGALLY_OBLIGATED | HIGH |
| `confirmed_viewing` AND rec_desc NOT in LEGALLY_OBLIGATED | MODERATE |
| Mitigation in G3_HIGH | HIGH |
| Any other mitigation | HIGH (default) |

**G3_EXTREME** = `improper_use`, `unknown_mit`, `sent_to_media`, `ssn_exposed`  
**G3_HIGH** = `unable_retrieve`, `confirmed_viewing`, `ransomware_confirmed`, `unsure_backup`  
**G3_MODERATE** = *(empty — all three prior members moved to G3_HIGH in batches 9 and 10)*

**Why G3_MODERATE is now empty:** `confirmed_viewing` and `ransomware_confirmed` were moved to G3_HIGH in batch 9. `unsure_backup` was moved to G3_HIGH in batch 10 (PT068/PT069). All represent known or probable access rather than pure uncertainty.

**Why unauthorized + unintentional gives HIGH for LEGALLY_OBLIGATED recipients:** When data accidentally reaches someone in an organizationally obligated role (OHCA, covered entity, etc.) — even as an unauthorized recipient — their statutory duties provide some mitigation even without sufficient disposition. PT062 confirmed this.

**Why `ransomware_confirmed` + malicious escalates to EXTREME for authorized:** PT065 (federal agency + malicious + ransomware_confirmed) → EXTREME. When intent was malicious AND ransomware acquired the data, even an authorized recipient context warrants EXTREME — the data was intentionally exfiltrated.

---

### Gate 4 — Fallthrough

**Result: MODERATE** — has never fired in 82 verified cases.

---

## Mitigation Buckets — Complete Reference

### Gate 1a (CONFIRMED_SAFE)
`unopened`, `forensic`, `backup`, `no_retain`  
Plus: `satisfactory` when `recip_type = authorized` AND rec_desc in LEGALLY_OBLIGATED  
Plus: `no_written_obtained` when `recip_type = authorized` AND rec_desc in {`covered_entity`, `federal_agency`} AND `nature < 0.45`

### Gate 1b (ASSURED_SAFE)
`permitted`, `written_obtained`, `written_provided`, `satisfactory` (non-obligated or unauthorized),  
`obligated_no_assurance`, `ce_ba_obligated`, `no_written_obtained` (fallthrough), `no_written_provided`, `company_policy`

Plain text + authorized + **strong** mits (`permitted`, `written_obtained`, `written_provided`, `satisfactory`, `ce_ba_obligated`, `company_policy`) → LOW  
Plain text + authorized + **weak** mits (`obligated_no_assurance`, `no_written_obtained`, `no_written_provided`) → MODERATE  
Non-plain-text + authorized → MODERATE regardless

### Gate 3 sub-buckets

| Bucket | Keys | Result (authorized) |
|---|---|---|
| G3_EXTREME | `improper_use`, `unknown_mit`, `sent_to_media`, `ssn_exposed` | EXTREME |
| G3_HIGH | `unable_retrieve`, `confirmed_viewing`*, `ransomware_confirmed`**, `unsure_backup` | HIGH |
| G3_MODERATE | *(empty)* | — |

*`confirmed_viewing` → MODERATE for non-LEGALLY_OBLIGATED authorized recipients  
**`ransomware_confirmed` → EXTREME when nature ≥ 0.80

### Special recipient categories (Gate 1b)

| Category | Key | Behavior |
|---|---|---|
| Employee-like | `employee` | Authorized rules: malicious → MODERATE, else → LOW |
| Attorney-like | `attorney` | Extreme tier: malicious → EXTREME, else → HIGH |
| Hacker/Media | `hacker`, `media` | Protection + intent table (see Gate 1b) |

### LEGALLY_OBLIGATED
`covered_entity`, `business_associate`, `ba_state_gov`, `federal_agency`, `ohca`,  
`self_insured_sponsor`, `fully_insured_sponsor`, `reg_investment_advisor`, `state_gov`, `employee`

---

## Version History

| Version | Cases | Key change |
|---|---|---|
| v1 | 0/3 | Weighted score model — wrong approach |
| v2 | 3/3 | Decision tree — mitigation as primary gate |
| v3 | 10/10 | Gate 2 de-id sub-rules |
| v4 | 19/19 | Gate 3 authorized vs unauthorized; incident nature in Gate 2 |
| v5 | 28/28 | Gate 1 split into 1a (confirmed) and 1b (assured) |
| v6 | 34/34 | `satisfactory` splits by legal obligation; Gate 1b by protection; Gate 2 mit sub-rules |
| v7 | 39/39 | Gate 3 authorized EXTREME/HIGH/MODERATE by mitigation |
| v8 | 43/43 | Ransomware + hacker/media exception on Gate 1a |
| v9 | 43/43 | Gate 1a satisfactory guard requires `recip_type = authorized`; Gate 1b dead code removed |
| v10 | 48/48 | `no_written_obtained` + authorized + obligated → LOW; Gate 1a exception broadened to `nature ≥ 0.80`; Gate 1b hacker/media escalation |
| v11 | 51/51 | Gate 1a exception to all unauthorized; Gate 1b: all unauthorized + malicious → HIGH |
| v12 | 54/54 | `confirmed_viewing` and `ransomware_confirmed` → G3_HIGH; S1/S4/S5 corrected; `unsure_backup` still MODERATE |
| v13 | 82/82 | 28 plain-text cases (batch 10): Gate 1a exception is mit-specific; `no_written_provided` removed from Gate 1a LOW; authorized Gate 1b split by protection level; hacker/media escalation protection-dependent; employee/attorney special rules; Gate 3 unauthorized+unintentional sub-rules; `unsure_backup` → G3_HIGH; G3_MODERATE now empty |

---

## Test Batches Summary

| Batch | Cases | Result | Key discovery |
|---|---|---|---|
| Original (O1–O3) | 3 | 0/3 → fixed | Weighted score wrong |
| Batch 1 (P1–P7) | 7 | 7/7 | Gate 2 structure confirmed |
| Batch 2 (N1–N9) | 9 | 9/9 | Authorized vs unauthorized split in Gate 3 |
| Batch 3 (Q1–Q9) | 9 | 9/9 | Gate 1a/1b split |
| Batch 4 (R1–R6) | 6 | 6/6 | `satisfactory` legal-obligation rule; Gate 2 refinement |
| Batch 5 (S1–S5) | 5 | 5/5 | Gate 3 three-tier split — **S1/S4/S5 corrected batch 9** |
| Batch 6 (T1–T4) | 4 | 4/4 | Ransomware+hacker/media exception on Gate 1a |
| Batch 7 (F1–F5) | 5 | 5/5 | `no_written_obtained` + obligated → LOW; Gate 1a broadened; Gate 1b hacker/media escalation |
| Batch 8 (V1–V3) | 3 | 3/3 | Gate 1b unauthorized+malicious=HIGH; Gate 1a broadened to all unauthorized |
| Batch 9 (C1–C3 + corrections) | 6 | 6/6 | `confirmed_viewing` and `ransomware_confirmed` → G3_HIGH |
| Batch 10 (PT003–PT069, 28 cases) | 28 | 28/28 | 9 rule corrections — see v13 changes above |
| **Total** | **82** | **82/82** | |

---

## Electronic Category — Score Assignments

The Category dropdown (dimension 1) was added to support both Paper and Electronic incidents. Selecting a category populates the Protection Measure dropdown with the appropriate options.

**Electronic protection measures and assigned scores:**

| Group | Option | Score | Analogy |
|---|---|---|---|
| Encrypted | Meets NIST standard, key not compromised, no evidence of access | 0.20 | Redacted |
| Encrypted | Unsure of NIST standard, key not compromised, no evidence of access | 0.25 | Under physical safeguard |
| Encrypted | No evidence of access but unsure of encryption key security | 0.30 | Limited data set |
| Encrypted | Encryption key was compromised | 0.85 | In plain text |
| Encrypted | Evidence of access with valid credentials | 0.85 | In plain text |
| Password protected | Password was not compromised | 0.25 | Under physical safeguard |
| Password protected | Password was compromised | 0.85 | In plain text |
| — | Statistically de-identified | 0.10 | Same as Paper |
| — | Redacted | 0.20 | Same as Paper |
| — | Limited data set | 0.30 | Same as Paper |
| — | Data is identifiable or can be re-identified | 0.85 | In plain text |
| — | No protection measures were present | 0.85 | In plain text |

Electronic cases are not yet empirically verified against Radar. See recommended test cases in the previous README section.

---

## N/A Mitigations (Not Shown in Radar for Certain Combinations)

Batch 10 identified two mitigation options that do not appear in Radar's UI for specific combinations:

- **`company_policy`** — not shown as an option for unauthorized recipients. Remove from test cases where recip_type = unauthorized.
- **`ssn_exposed`** — not shown for media recipients. Remove from test cases where rec_desc = media.

These are UI mapping gaps, not tree logic issues. The tree handles them correctly if they were selectable.

---

## File Structure

```
index.html                        <- single deployable file (GitHub Pages)
README.md                         <- this file
breach_risk_test_cases.xlsx       <- 54 verified cases, re-test checklist, batch summary
plain_text_radar_tests.xlsx       <- batch 10 plain text test cases with Radar results
risk_test_log_*.csv               <- exported test logs from each testing batch
Extracted_Mapping_Tables.xlsx     <- source of truth for dropdown cascade mappings
```

---

## How to Deploy on GitHub Pages

1. Ensure the file is named `index.html`
2. Push to a public GitHub repository
3. Go to **Settings → Pages → Source → main branch**

---

## Remaining Testing Priorities

1. **Electronic protection measures** — none yet verified against Radar. Run E1–E7 from previous README before treating Electronic results as reliable.
2. **`confirmed_viewing` + non-obligated authorized + malicious** — PT067 confirmed unintentional → MODERATE. Does malicious intent change this to HIGH? Needs testing.
3. **`backup` + unauthorized + unintentional** — PT006 (not-malicious) → MODERATE, but unintentional not tested.
4. **`no_retain` + unauthorized + malicious** — PT003 logic predicts MODERATE; untested.
5. **Gate 4** — has never fired. If it fires, a new Radar mitigation exists outside current buckets.

---

## Spring Boot Integration

```java
public String assessRisk(RiskInput i) {
  String mit=i.mitigationKey; double prot=i.protectionScore;
  double nat=i.natureScore; boolean auth="authorized".equals(i.recipientType);
  boolean unauth=!auth; String rec=i.recipientDesc;

  // Gate 1a
  boolean g1a = CONFIRMED_SAFE.contains(mit)
    || ("satisfactory".equals(mit) && auth && LEGALLY_OBLIGATED.contains(rec))
    || ("no_written_obtained".equals(mit) && auth && NO_WRITTEN_LOW_RECIPS.contains(rec) && nat<0.45);

  if(g1a) {
    if(unauth && prot>0.10) {
      if("forensic".equals(mit)) return "LOW";
      if("backup".equals(mit))   return nat>=0.80?"HIGH":"MODERATE";
      if("no_retain".equals(mit))return nat>=0.80?"MODERATE":"LOW";
      if("unopened".equals(mit)) return (isHackerMedia(rec)&&nat>=0.80)?"MODERATE":"LOW";
    }
    return "LOW";
  }

  // Gate 1b
  if(ASSURED_SAFE.contains(mit)) {
    if(prot<=0.10) return "LOW";
    if("employee".equals(rec))  return nat>=0.80?"MODERATE":"LOW";
    if(auth) return prot>=0.85?(AUTH_WEAK_MITS.contains(mit)?"MODERATE":"LOW"):"MODERATE";
    if("attorney".equals(rec))  return nat>=0.80?"EXTREME":"HIGH";
    if(isHackerMedia(rec)) {
      boolean lp=prot<=0.20;
      if(nat>=0.80) return lp?"EXTREME":"HIGH";
      if(nat<=0.30) return "HIGH";
      return lp?"HIGH":"MODERATE";
    }
    return nat>=0.80?"HIGH":"MODERATE";
  }

  // Gate 2
  if(prot<=0.10) return assessGate2(i);

  // Gate 3
  if(prot>0.10 && i.dispositionScore>=0.70) return assessGate3(i);

  return "MODERATE"; // Gate 4
}

private String assessGate3(RiskInput i) {
  if("unauthorized".equals(i.recipientType)) {
    if(i.natureScore<=0.30) {
      if(LEGALLY_OBLIGATED.contains(i.recipientDesc)) return "HIGH";
      if(G3_EXTREME.contains(i.mitigationKey)) return "EXTREME";
      return "HIGH";
    }
    return "EXTREME";
  }
  if(G3_EXTREME.contains(i.mitigationKey)) return "EXTREME";
  if("ransomware_confirmed".equals(i.mitigationKey)&&i.natureScore>=0.80) return "EXTREME";
  if("confirmed_viewing".equals(i.mitigationKey))
    return LEGALLY_OBLIGATED.contains(i.recipientDesc)?"HIGH":"MODERATE";
  if(G3_HIGH.contains(i.mitigationKey)) return "HIGH";
  return "HIGH";
}

// v13 bucket constants
static final Set<String> G3_HIGH = Set.of(
  "unable_retrieve","confirmed_viewing","ransomware_confirmed","unsure_backup");
// G3_MODERATE is empty in v13
static final Set<String> AUTH_WEAK_MITS = Set.of(
  "obligated_no_assurance","no_written_obtained","no_written_provided");
static final Set<String> NO_WRITTEN_LOW_RECIPS = Set.of("covered_entity","federal_agency");
```
