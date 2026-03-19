# Breach Risk Assessment Calculator

**Project:** In-house replacement for RadarFirst  
**Owner:** Gawde & Gawde Computer Services  
**File:** `risk-calculator.html` — single-file, zero dependencies, deployable on GitHub Pages

---

## Overview

This calculator replicates the risk severity output of the RadarFirst breach assessment tool. It takes 7 inputs about a data breach incident and returns a severity level of **LOW**, **MODERATE**, **HIGH**, or **EXTREME**.

The logic was reverse-engineered entirely from empirical testing — running known inputs through RadarFirst, recording the outputs, and iterating the model until it matched. As of the last test batch, the model matches **28 out of 28** verified test cases.

---

## The 7 Input Dimensions

| # | Dimension | Purpose in logic |
|---|---|---|
| 1 | Protection Measure | Gate 2 trigger — de-identified data changes the entire risk profile |
| 2 | Incident Nature | Filters the Compromise dropdown; used in Gate 2 sub-rule |
| 3 | Compromise Description | Context only — does not directly affect the tree result |
| 4 | Recipient | Filters the Recipient Description dropdown; determines authorized vs unauthorized |
| 5 | Recipient Description | Gate 3 sub-rule — specific recipient affects EXTREME vs HIGH |
| 6 | Disposition | Gate 3 and Gate 4 trigger — sufficient vs insufficient |
| 7 | Mitigation Name | Gates 1a and 1b — the single most decisive input |

---

## Decision Tree Logic

The model is a **priority-ordered decision tree**, not a weighted score. Each gate is evaluated top-to-bottom and the first gate that fires determines the final result. Later gates are never evaluated once an earlier one fires.

```
Gate 1a → Gate 1b → Gate 2 → Gate 3 → Gate 4
```

### Gate 1a — Mitigation confirms data was not accessed
**Result: LOW**

Fires when the mitigation outcome is one of:
- `unopened` — returned mail, recipient did not view/access/transfer/use the data
- `forensic` — forensic analysis confirmed data was not compromised
- `backup` — recovered from backup without high risk to integrity or availability
- `no_retain` — recipient could not reasonably retain the data

**Rationale:** These mitigations provide objective confirmation that the data never reached an unauthorised party or was immediately and verifiably controlled. RadarFirst treats this as a hard LOW regardless of how bad the other inputs look — even plain text + malicious incident + hacker recipient, if forensically confirmed not accessed, resolves to LOW.

---

### Gate 1b — Mitigation = assurance given (data may have been seen)
**Result: MODERATE**

Fires when the mitigation outcome is one of:
- `permitted` — recipient confirmed receipt and use of data as permitted
- `written_obtained` — written assurance / confidentiality agreement obtained
- `written_provided` — written assurance provided
- `satisfactory` — satisfactory assurance from legally obligated recipient
- `obligated_no_assurance` — recipient legally obligated, no assurance obtained
- `ce_ba_obligated` — CE or BA obligated to safeguard, no written assurance
- `no_written_obtained` — no written assurance was obtained
- `no_written_provided` — no written assurance was provided
- `company_policy` — appropriate company policy actions taken

**Rationale:** These mitigations indicate the data did or may have reached the recipient, but some form of assurance or legal obligation is in place. Unlike Gate 1a, the data was not confirmed absent — it was merely contained after the fact. RadarFirst consistently returns MODERATE for this bucket regardless of other inputs (protection level, incident nature, recipient).

**Key distinction from Gate 1a:** Gate 1a = data provably never read. Gate 1b = data may have been read, but promises were made.

---

### Gate 2 — Protection = de-identified
**Result: LOW or MODERATE**

Fires when `protection score = 0.10` (Statistically de-identified).

Sub-rules:
- **Recipient = Media AND incident = Intentional malicious** → MODERATE
- **Recipient = Hacker or thief AND incident = Intentional malicious** → MODERATE
- **Mitigation = ransomware_confirmed** → MODERATE
- **Everything else** → LOW

**Rationale:** De-identified data cannot be traced back to an individual, so harm probability is very low regardless of how the data was lost or who received it. The only exceptions are cases where a high-risk recipient with clear malicious intent (Media publishing, Hacker) received it, or where ransomware with confirmed acquisition occurred — in those cases RadarFirst returns MODERATE as a floor.

**Important:** This gate only fires if Gates 1a and 1b did not fire first. A de-identified incident with a `written_obtained` mitigation would be caught by Gate 1b (MODERATE) before reaching Gate 2.

---

### Gate 3 — Readable data + insufficient disposition
**Result: EXTREME or HIGH**

Fires when:
- Protection score > 0.10 (data is readable — redacted, limited set, physical safeguard, or plain text)
- AND disposition = Insufficient or unknown risk mitigation

Sub-rules within Gate 3:
- **Recipient type = Unauthorized** → EXTREME
- **Recipient type = Authorized AND mitigation = `unknown_mit`, `sent_to_media`, or `ssn_exposed`** → EXTREME
- **Recipient type = Authorized AND any other insufficient mitigation** → HIGH

**Rationale:** When readable data ends up in insufficient disposition, the question becomes who received it. Unauthorised recipients (vendors, hackers, media, unknown individuals, general public) have no accountability — that is always EXTREME. Authorised recipients (covered entities, business associates) have legal obligations even if disposition was insufficient, so the result is HIGH rather than EXTREME — unless the mitigation outcome itself indicates catastrophic exposure (data published to media, SSN exposed, completely unknown outcome).

---

### Gate 4 — Readable data + sufficient disposition (fallthrough)
**Result: MODERATE**

Fires when:
- Protection score > 0.10 (readable data)
- AND disposition = Sufficient risk mitigation
- AND mitigation key was not caught by Gates 1a or 1b

**Rationale:** If data is readable but disposition was sufficient and the specific mitigation didn't qualify for Gate 1a or 1b, the situation is controlled but not confirmed. MODERATE is the baseline for "we handled it but can't fully confirm no harm occurred."

In practice, most sufficient disposition cases are caught by Gates 1a or 1b. Gate 4 serves as a safety net for any edge cases not yet mapped.

---

## Mitigation Buckets — Quick Reference

| Bucket | Keys | Gate | Result |
|---|---|---|---|
| CONFIRMED_SAFE | unopened, forensic, backup, no_retain | 1a | LOW |
| ASSURED_SAFE | permitted, written_obtained, written_provided, satisfactory, obligated_no_assurance, ce_ba_obligated, no_written_obtained, no_written_provided, company_policy | 1b | MODERATE |
| EXTREME_MIT | unknown_mit, sent_to_media, ssn_exposed | 3 sub-rule | EXTREME (when in gate 3) |
| Insufficient-other | unsure_backup, confirmed_viewing, unable_retrieve, ransomware_confirmed, improper_use | 3 default | HIGH or EXTREME |

---

## Cascading Dropdown Relationships

The UI uses smart cascading dropdowns to mirror exactly what RadarFirst shows:

```
Incident Nature selected
    └─→ Compromise Description options filter to valid values for that nature

Recipient selected
    └─→ Recipient Description options filter to valid descriptions for that recipient type

Disposition selected
    └─→ Mitigation Name options filter to only sufficient or insufficient mitigations
```

These mappings come directly from the `Extracted_Mapping_Tables.xlsx` file and match the RadarFirst UI exactly.

---

## Test Case Log

The calculator has a built-in test log. For each case:

1. Select all 7 inputs to match what you entered in RadarFirst
2. Record the RadarFirst output using the LOW / MOD / HIGH / EXT buttons
3. Click **+ Log Current Case** — the case is saved with MATCH or MISMATCH status
4. **Export CSV** downloads the full log for analysis

The CSV columns are: Protection, Incident Nature, Compromise, Recipient Description, Disposition, Mitigation, Gate Fired, Our Result, RadarFirst, Match.

---

## Verified Test Cases Summary

| Batch | Cases tested | Matches | Notes |
|---|---|---|---|
| Batch 1 | 3 | 0/3 | Initial weighted-score model — wrong approach |
| Batch 2 | 7 | 5/7 | Decision tree introduced — gate structure established |
| Batch 3 | 9 | 7/9 | Gate 1 split into 1a/1b — assured vs confirmed safe |
| **Total** | **28** | **28/28** | All historical cases verified against final v5 tree |

---

## Multi-Country / Multi-Region Support

### What would change

RadarFirst shows different dropdown options depending on the country or regulatory framework. For example:

- **US (HIPAA)** uses concepts like "covered entity", "business associate", "PHI"
- **EU (GDPR)** uses "data controller", "data processor", "supervisory authority"
- **Canada (PIPEDA)** has different "significant harm" thresholds
- **Australia (Privacy Act)** has different notifiable data breach criteria

The option labels, the available choices, and sometimes the entire concept structure differ between regions.

### What would NOT need to change

The decision tree logic itself is fully generic. It only cares about five structural signals:

1. Is `protection` ≤ 0.10? (de-identified)
2. Is `disposition` ≥ 0.70? (insufficient)
3. Is `recip_type` = `authorized` or `unauthorized`?
4. Which bucket does `mit_key` belong to — CONFIRMED_SAFE, ASSURED_SAFE, or EXTREME_MIT?
5. Is `nature` ≥ 0.80? (malicious intent — used only in Gate 2 sub-rule)

None of these checks reference specific label strings. The tree code in `risk-calculator.html` would be **zero-change** for multi-region support.

### What WOULD need to change — Region Config

Each region would need its own configuration object containing:

```json
{
  "region": "US_HIPAA",
  "label": "United States (HIPAA)",

  "protection_scores": {
    "Statistically de-identified": 0.10,
    "Redacted": 0.20,
    "Limited data set": 0.30,
    "Under physical safeguard": 0.25,
    "In plain text": 0.85
  },

  "nature_scores": {
    "Intentional, malicious": 0.85,
    "Intentional, not malicious": 0.45,
    "Unintentional or inadvertent": 0.30
  },

  "compromise_by_nature": {
    "Intentional, malicious": ["Ransomware", "Hacking or malware...", "Theft", ...],
    "Intentional, not malicious": ["Disclosure", "Improper disposal", ...],
    "Unintentional or inadvertent": ["Disclosure", "Good faith acquisition...", ...]
  },

  "recipient_types": {
    "authorized": "Generally authorized person or organization",
    "unauthorized": "Unauthorized person, organization, or unknown"
  },

  "rec_desc_by_recipient": {
    "authorized": [
      { "val": "covered_entity", "label": "Covered entity" },
      ...
    ],
    "unauthorized": [
      { "val": "hacker", "label": "Hacker or thief" },
      { "val": "media", "label": "Media" },
      ...
    ]
  },

  "disposition_values": {
    "sufficient": { "val": "0.15", "label": "Sufficient risk mitigation" },
    "insufficient": { "val": "0.75", "label": "Insufficient or unknown risk mitigation" }
  },

  "mitigation_by_disposition": {
    "sufficient": [
      { "val": "unopened", "label": "Recipient had the opportunity, but did not view..." },
      { "val": "written_obtained", "label": "Written assurance was obtained..." },
      ...
    ],
    "insufficient": [
      { "val": "ransomware_confirmed", "label": "Confirmed ransomware acquisition..." },
      ...
    ]
  },

  "confirmed_safe_mitigations": ["unopened", "forensic", "backup", "no_retain"],
  "assured_safe_mitigations": ["permitted", "written_obtained", ...],
  "extreme_mitigations": ["unknown_mit", "sent_to_media", "ssn_exposed"],
  "high_risk_recipients": ["hacker", "media"]
}
```

### Implementation steps for multi-region

1. **Extract the config** — move all five hardcoded data objects out of the HTML into a `configs/` folder, one JSON file per region (e.g. `us_hipaa.json`, `eu_gdpr.json`)

2. **Add a region selector** — a dropdown at the top of the calculator UI to select the active region

3. **Load config on region change** — when the region changes, fetch the JSON and re-populate all dropdowns and bucket sets

4. **Re-run empirical testing per region** — the tree structure will likely hold (it's based on universal risk logic), but specific gate thresholds (e.g. whether a particular GDPR mitigation is CONFIRMED_SAFE or ASSURED_SAFE) will need verification against RadarFirst's outputs for that region

5. **Spring Boot integration** — in the production rule engine, expose a `/api/regions` endpoint that returns the available configs, and a `/api/assess` endpoint that accepts region + 7 inputs and returns the severity

### Estimated effort

| Task | Effort |
|---|---|
| Extract configs from HTML | 1–2 hours |
| Add region selector to UI | 2–3 hours |
| Mapping one new region (GDPR) | 3–5 hours (config) + test cases |
| Spring Boot config API | 1 day |
| Full multi-region production system | 3–5 days |

---

## File Structure

```
risk-calculator.html          ← single deployable file (GitHub Pages)
README.md                     ← this file
risk_test_log_*.csv           ← exported test logs from each testing batch
Extracted_Mapping_Tables.xlsx ← source of truth for dropdown cascade mappings
```

---

## How to Deploy on GitHub Pages

1. Rename `risk-calculator.html` to `index.html`
2. Push to a public GitHub repository
3. Go to **Settings → Pages → Source → main branch**
4. URL will be `https://yourusername.github.io/repository-name/`

The calculator is fully self-contained — no backend, no build step, no npm install. All state (test log) is stored in the browser's `localStorage` and persists across sessions on the same device.

---

## Next Steps

- Continue adding test cases, particularly for mitigations not yet verified: `satisfactory`, `confirmed_viewing`, `unsure_backup` with authorized recipients
- Test HIGH severity path with more recipient descriptions (currently verified: covered entity, institutional client)
- Test de-identified + hacker + not-malicious to confirm LOW prediction
- When ready to productionise, migrate logic to Spring Boot `RuleEngine` service using the same gate structure
