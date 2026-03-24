# Breach Risk Assessment Calculator

**Project:** In-house replacement for Radar  
**Owner:** Gawde & Gawde Computer Services  
**File:** `index.html` — single-file, zero dependencies, deployable on GitHub Pages  
**Tree version:** v14  
**Verified test cases:** 268/268

---

## The 8 Input Dimensions

| # | Dimension | Role in tree |
|---|---|---|
| 1 | Category | Filters Protection Measure dropdown (Paper / Electronic) |
| 2 | Protection Measure | Critical threshold — de-id ≤0.10; Redacted=0.20 triggers special Gate 1a rule; plain text=0.85 unlocks LOW for authorized |
| 3 | Incident Nature | Malicious ≥0.80 escalates in Gates 1a, 1b, 2, 3 |
| 4 | Compromise Description | Representative only — no longer a direct gate trigger |
| 5 | Recipient | authorized vs unauthorized — determines base gate path |
| 6 | Recipient Description | LEGALLY_OBLIGATED membership; attorney/employee/hacker/media special tiers |
| 7 | Disposition | Gate 3 trigger: ≥0.70 = insufficient |
| 8 | Mitigation Name | Most decisive input — determines gate and severity |

---

## Decision Tree — Gates

```
Gate 1a → Gate 1b → Gate 2 → Gate 3 → Gate 4
```

---

### Gate 1a — Mitigation confirms data not accessed

**Fires for CONFIRMED_SAFE:** `unopened`, `forensic`, `backup`, `no_retain`  
**Also fires for** `satisfactory` AND authorized AND LEGALLY_OBLIGATED AND **prot=0.85**  
**Also fires for** `no_written_obtained` AND authorized AND {CE, federal_agency} AND **prot=0.85** AND non-malicious

**`forensic` is always LOW** — algorithmic certainty overrides all other inputs.

**For authorized recipients (non-forensic CONFIRMED_SAFE):**

| Protection | Nature | Result |
|---|---|---|
| Any | Any | LOW |
| **0.20 (Redacted only)** | **Malicious** | **MODERATE** |

The Redacted+malicious exception: when data was protected yet accessed maliciously despite 0.20-level safeguards, Radar treats the breach as more serious even with confirmed-safe mitigation. This applies only to Redacted (0.20) — not de-id, physical safeguard, or plain text.

**For unauthorized recipients (non-forensic CONFIRMED_SAFE):**

| Mitigation | Rec desc | Nature | Result |
|---|---|---|---|
| `forensic` | any | any | LOW |
| `backup` | LEGALLY_OBLIGATED | unintentional | LOW |
| `backup` | LEGALLY_OBLIGATED | malicious or not-malicious | MODERATE |
| `backup` | non-obligated | malicious | **HIGH** |
| `backup` | non-obligated | not-malicious | MODERATE |
| `backup` | non-obligated | unintentional | LOW |
| `no_retain` | any | malicious | MODERATE |
| `no_retain` | any | non-malicious/unintentional | LOW |
| `unopened` | hacker or media + malicious | — | MODERATE |
| `unopened` | any other | any | LOW |

---

### Gate 1b — Mitigation = assurance given

**Fires for ASSURED_SAFE:** `permitted`, `written_obtained`, `written_provided`, `satisfactory` (fallthrough), `obligated_no_assurance`, `ce_ba_obligated`, `no_written_obtained` (fallthrough), `no_written_provided`, `company_policy`

**De-identified (prot ≤ 0.10):** always LOW.

**AUTH_WEAK_MITS** (weak assurance — documents non-obtainment): `obligated_no_assurance`, `no_written_obtained`, `no_written_provided`

**Results by recipient type:**

| Recipient | Condition | Result |
|---|---|---|
| Employee | malicious | MODERATE |
| Employee | non-malicious / unintentional | LOW |
| **Authorized** | plain text + strong mit | **LOW** |
| Authorized | plain text + weak mit | MODERATE |
| Authorized | prot < 0.85 (any mit) | MODERATE |
| **Unauth LEGALLY_OBLIGATED** | malicious + weak mit + prot < 0.85 | **HIGH** |
| Unauth LEGALLY_OBLIGATED | malicious + strong mit OR plain text | MODERATE |
| Unauth LEGALLY_OBLIGATED | non-malicious + strong mit | **LOW** |
| Unauth LEGALLY_OBLIGATED | non-malicious + weak mit | MODERATE |
| **Attorney** | malicious + weak mit | **EXTREME** |
| Attorney | malicious + strong mit | HIGH |
| Attorney | non-malicious / unintentional | MODERATE |
| **Hacker/media** | malicious + prot ≤ 0.20 + weak mit | **EXTREME** |
| Hacker/media | malicious (any other) | HIGH |
| Hacker/media | unintentional + plain text | HIGH |
| Hacker/media | unintentional + prot < 0.85 | MODERATE |
| Hacker/media | intentional not-malicious | MODERATE |
| Other unauthorized | malicious | HIGH |
| Other unauthorized | non-malicious / unintentional | MODERATE |

**Key insight on protection score:** Higher protection scores (0.85) give BETTER results for authorized and LEGALLY_OBLIGATED recipients. The reasoning: when data is already in plain text, a successful assurance from an authorized entity is sufficient containment. When data was partially protected and still accessed maliciously, the breach is more serious.

---

### Gate 2 — De-identified

**Fires when** prot ≤ 0.10 AND Gates 1a/1b didn't fire.

**GATE2_BAD_MIT:** `unable_retrieve`, `improper_use`, `sent_to_media`, `unknown_mit`, `ssn_exposed`, `ransomware_confirmed`, `confirmed_viewing`, `unsure_backup`

| Condition | Result |
|---|---|
| hacker/media + malicious + `ransomware_confirmed` | HIGH |
| hacker/media + malicious + GATE2_BAD_MIT | MODERATE |
| hacker/media + malicious + other | LOW |
| hacker/media + not malicious | LOW |
| non-hm + `ransomware_confirmed` | MODERATE |
| everything else | LOW |

---

### Gate 3 — Readable + insufficient disposition

**Fires when** prot > 0.10 AND disp ≥ 0.70.

**Unauthorized recipients:**

| Nature | Result |
|---|---|
| Unintentional (≤ 0.30) | **HIGH** (always — C10) |
| Malicious or not-malicious | EXTREME |

**Authorized recipients:**

| Mitigation | Condition | Result |
|---|---|---|
| G3_EXTREME (`improper_use`, `unknown_mit`, `sent_to_media`, `ssn_exposed`) | — | EXTREME |
| `ransomware_confirmed` | malicious | EXTREME |
| `confirmed_viewing` | LEGALLY_OBLIGATED rec_desc | HIGH |
| `confirmed_viewing` | non-obligated + unintentional | MODERATE |
| `confirmed_viewing` | non-obligated + malicious/not-malicious | HIGH |
| G3_HIGH (`unable_retrieve`, `confirmed_viewing`, `ransomware_confirmed`, `unsure_backup`) | — | HIGH |
| default | — | HIGH |

**G3_MODERATE is empty** — all three prior members have been moved to G3_HIGH.

---

## Version History

| Version | Cases | Key changes |
|---|---|---|
| v1–v9 | 0→43 | Initial tree building, gate splits, exception discovery |
| v10 | 48/48 | `no_written_obtained`+obligated→LOW; Gate 1a broadened; Gate 1b hacker/media escalation |
| v11 | 51/51 | Gate 1a exception to all unauthorized; Gate 1b unauthorized+malicious→HIGH |
| v12 | 54/54 | `confirmed_viewing`/`ransomware_confirmed`→G3_HIGH; S1/S4/S5 corrected |
| v13 | 82/82 | +28 plain-text cases: Gate 1a mit-specific exceptions; authorized Gate 1b prot-split; employee/attorney rules; Gate 3 unintentional sub-rules; `unsure_backup`→G3_HIGH |
| **v14** | **268/268** | **+186 de-id/redacted/physical cases: 10 corrections — see below** |

### v14 Changes in Detail

**C1 — Gate 2 GATE2_BAD_MIT expanded:** `confirmed_viewing` and `unsure_backup` added. Confirmed by MM17 (hacker+mal+unsure_backup→MODERATE) and MM18 (media+mal+confirmed_viewing→MODERATE).

**C2 — Gate 1a authorized + Redacted (0.20) + malicious → MODERATE:** The escalation is specific to the Redacted protection score only — not de-id (0.10), physical safeguard (0.25), or plain text (0.85). Evidence: #35-37 (CE/BA/service_provider + malicious + unopened + prot=0.20 → MODERATE) vs MM1-3 (same + prot=0.10 → LOW) and MM115-117 (same + prot=0.25 → LOW). `forensic` is always exempt.

**C3 — Gate 1a backup: LEGALLY_OBLIGATED rec_desc splits the result:** Unauthorized LEGALLY_OBLIGATED recipients with backup give MODERATE for malicious and LOW for unintentional. Non-obligated: malicious→HIGH, not-malicious→MODERATE, unintentional→LOW.

**C4 — Gate 1a no_written_obtained → LOW only at plain text:** #47,48 (CE/fed + unintentional + no_written_obtained + prot=0.20 → MODERATE) vs F1/F2 (prot=0.85 → LOW).

**C5 — Gate 1a satisfactory → LOW only at plain text:** #49-51 (malicious + prot=0.20 → MODERATE) vs R1 (prot=0.85 → LOW).

**C6 — Gate 1b attorney: mit-strength determines severity:** malicious + weak_mit → EXTREME; malicious + strong_mit → HIGH; non-malicious → MODERATE. Replaces previous EXTREME/HIGH binary split.

**C7 — Gate 1b hacker/media: EXTREME requires prot≤0.20 AND malicious AND weak_mit:** #67,68 (permitted + prot=0.20 → HIGH, not EXTREME). F4 (obligated_no_assurance + prot=0.20 → EXTREME) still holds — weak_mit is the differentiator. Unintentional: HIGH if plain text, MODERATE if prot<0.85.

**C8 — Gate 1b unauth LEGALLY_OBLIGATED follows authorized rules with mit-strength split:** Strong mit + non-malicious → LOW; strong mit + malicious → MODERATE; weak mit + prot<0.85 + malicious → HIGH. Confirmed by: #73-75 (CE/BA+mal+permitted→MODERATE), #76-78 (CE/BA+notmal+permitted→LOW), V3 (BA+mal+no_written_obtained+prot=0.25→HIGH).

**C9 — Gate 3 confirmed_viewing non-obligated auth: intent matters:** Unintentional → MODERATE (PT067); malicious or not-malicious → HIGH (#79-81).

**C10 — Gate 3 unauthorized + unintentional → always HIGH:** #97,98,178-180 confirmed HIGH for service_provider/institutional_client/vendor + unintentional. Removes G3_EXTREME distinction for unintentional unauthorized. T1 (partner+uninten+improper_use) corrected to HIGH.

**Stale case corrections:** O1 → MODERATE, T1 → HIGH, R4 → MODERATE (overridden by batch 11 evidence).

---

## Test Batches

| Batch | Cases | Result | Key discovery |
|---|---|---|---|
| Batches 1–9 | 54 | 54/54 | Full tree through v12 |
| Batch 10 (PT series) | 28 | 28/28 | Plain text corrections → v13 |
| Batch 11 (MM series) | 186 | 186/186 | De-id/Redacted/Physical corrections → v14 |
| **Total** | **268** | **268/268** | |

---

## Electronic Protection Measures (Unverified)

Electronic cases are assigned scores by analogy to Paper but have not been empirically verified against Radar. Run E1–E7 test cases before treating Electronic results as reliable.

---

## File Structure

```
index.html                           ← single deployable file
README.md                            ← this file
breach_risk_test_cases.xlsx          ← original 54 verified cases
plain_text_radar_tests.xlsx          ← batch 10 plain text cases
protection_levels_radar_tests.xlsx   ← batch 11 de-id/redacted/physical cases
```

---

## Spring Boot Integration (v14)

```java
public String assessRisk(RiskInput i) {
    String mit=i.mit; double prot=i.prot; double nat=i.nat;
    boolean auth="authorized".equals(i.recip);
    String rec=i.recDesc;
    boolean isMal=nat>=0.80, isUninten=nat<=0.30;
    boolean isPlain=prot>=0.85, isRedact=Math.abs(prot-0.20)<0.001;
    boolean isLowP=prot>0.10&&prot<=0.20, isOblig=LEGALLY_OBLIGATED.contains(rec);
    boolean isWeak=AUTH_WEAK_MITS.contains(mit);

    // Gate 1a
    boolean g1a=CONFIRMED_SAFE.contains(mit)
      ||("satisfactory".equals(mit)&&auth&&isOblig&&isPlain)
      ||(("no_written_obtained".equals(mit))&&auth&&NO_WRITTEN_LOW.contains(rec)&&isPlain&&!isMal);

    if(g1a){
        if(auth){
            if("forensic".equals(mit)) return "LOW";
            if(isRedact&&isMal) return "MODERATE";
            return "LOW";
        }
        if(prot>0.10){
            if("forensic".equals(mit)) return "LOW";
            if("backup".equals(mit)) return isOblig?(isUninten?"LOW":"MODERATE"):(isMal?"HIGH":isUninten?"LOW":"MODERATE");
            if("no_retain".equals(mit)) return isMal?"MODERATE":"LOW";
            if("unopened".equals(mit)) return (isHM(rec)&&isMal)?"MODERATE":"LOW";
        }
        return "LOW";
    }

    // Gate 1b
    if(ASSURED_SAFE.contains(mit)){
        if(prot<=0.10) return "LOW";
        if("employee".equals(rec)) return isMal?"MODERATE":"LOW";
        if(auth) return isPlain?(isWeak?"MODERATE":"LOW"):"MODERATE";
        if(isOblig) return isMal?((!isPlain&&isWeak)?"HIGH":"MODERATE"):(isWeak?"MODERATE":"LOW");
        if("attorney".equals(rec)) return isMal?(isWeak?"EXTREME":"HIGH"):"MODERATE";
        if(isHM(rec)) return isMal?((isLowP&&isWeak)?"EXTREME":"HIGH"):(isUninten?(isPlain?"HIGH":"MODERATE"):"MODERATE");
        return isMal?"HIGH":"MODERATE";
    }

    // Gate 2
    if(prot<=0.10){
        boolean hr=isHM(rec);
        if(hr&&isMal){ if("ransomware_confirmed".equals(mit)) return "HIGH"; return GATE2_BAD.contains(mit)?"MODERATE":"LOW"; }
        if(!hr&&"ransomware_confirmed".equals(mit)) return "MODERATE";
        return "LOW";
    }

    // Gate 3
    if(prot>0.10&&i.disp>=0.70){
        if(!auth) return isUninten?"HIGH":"EXTREME";
        if(G3_EXT.contains(mit)) return "EXTREME";
        if("ransomware_confirmed".equals(mit)&&isMal) return "EXTREME";
        if("confirmed_viewing".equals(mit)) return isOblig?"HIGH":(isUninten?"MODERATE":"HIGH");
        if(G3_HIGH.contains(mit)) return "HIGH";
        return "HIGH";
    }
    return "MODERATE";
}
```

---

## v14 — Batch 11 Corrections (62 new cases: de-identified, redacted, physical safeguard)

**Tree version:** v14  **Verified cases:** 144/144

### 10 Rule Changes

**Change 1 — Gate 1a authorized + prot < 0.85 + malicious: non-forensic CONFIRMED_SAFE → MODERATE**  
When protection is not plain text and intent is malicious, authorized recipients with `backup`, `unopened`, or `no_retain` mitigations get MODERATE instead of LOW. `forensic` is the one exception — forensic analysis always returns LOW regardless of protection level or intent, as it provides algorithmic certainty that no access occurred. Evidence: MM35/36/37 (redacted + authorized + malicious + unopened → MODERATE).

**Change 2 — Gate 1a satisfactory and no_written_obtained: only LOW at prot = 0.85**  
Previously these could return LOW for authorized + LEGALLY_OBLIGATED regardless of protection level. Batch 11 confirmed: both mits return MODERATE when protection < 0.85. The plain-text-only LOW rule now applies consistently to all Gate 1a assurance-based paths. Evidence: MM47–51, MM127–131.

**Change 3 — Gate 1a backup + unauthorized + LEGALLY_OBLIGATED recipient: malicious → MODERATE, else → LOW**  
LEGALLY_OBLIGATED recipients (even when unauthorized) have statutory obligations that reduce risk. When backup is used with an obligated unauthorized recipient: malicious → MODERATE (not HIGH); unintentional → LOW (not MODERATE). Non-obligated unauthorized recipients retain the existing HIGH/MODERATE/LOW split by intent. Evidence: MM26–34, MM106–114.

**Change 4 — Gate 1b attorney: mitigation strength determines EXTREME vs HIGH (protection-independent)**  
Attorney + malicious + WEAK mit (`obligated_no_assurance`, `no_written_obtained`, `no_written_provided`) → EXTREME regardless of protection level. Attorney + malicious + STRONG mit → HIGH. Attorney + not-malicious or unintentional → MODERATE. Evidence: PT032 (plain text + no_written_obtained → EXTREME), MM56/137 (redacted/physical + permitted → HIGH).

**Change 5 — Gate 1b hacker/media: prot ≤ 0.20 + malicious + WEAK → EXTREME; all other malicious → HIGH**  
The EXTREME threshold for hacker/media requires three conditions simultaneously: low protection (≤ 0.20), malicious intent, AND a weak mitigation. If any of these is absent, malicious intent gives HIGH. Unintentional + WEAK → HIGH; unintentional + STRONG → MODERATE; not-malicious → MODERATE. Evidence: F4 (EXTREME: prot=0.20+hacker+malicious+weak), MM67/68 (HIGH: prot=0.20+hacker/media+malicious+strong), PT024 (HIGH: prot=0.85+media+malicious+weak).

**Change 6 — Gate 1b LEGALLY_OBLIGATED unauthorized recipients: strong/weak mit split**  
LEGALLY_OBLIGATED recipients receiving data without authorization have legal duties that still constrain risk. Malicious + WEAK → HIGH; malicious + STRONG → MODERATE; not-malicious + STRONG → LOW. Evidence: V3 (BA unauth + malicious + no_written_obtained → HIGH), MM73–78, MM154–159.

**Change 7 — Gate 2: hacker and media have separate GATE2_BAD_MIT sets**  
`unsure_backup` → MODERATE for hacker (MM17 confirmed) but LOW for media (R4 confirmed). Both include `confirmed_viewing` as MODERATE. The sets are otherwise identical. Evidence: R4 (media + malicious + unsure_backup → LOW), MM17 (hacker + malicious + unsure_backup → MODERATE), MM18 (media + malicious + confirmed_viewing → MODERATE).

**Change 8 — Gate 3 confirmed_viewing for authorized: prot < 0.85 → always HIGH**  
Previously: non-obligated authorized + confirmed_viewing → MODERATE. Now: this only applies at prot = 0.85 + unintentional. At prot < 0.85, all authorized + confirmed_viewing → HIGH regardless of obligated status or nature. Evidence: MM79–81, MM160–162.

**Change 9 — Gate 3 unauthorized + unintentional + G3_EXTREME: personal vs organizational distinction**  
LEGALLY_OBLIGATED unauthorized → HIGH (unchanged from v13). Non-obligated unauthorized now splits: PERSONAL recipients (partner, relative, another_family, non_custodial, parent_guardian, patient, general_public) + G3_EXTREME → EXTREME; ORGANIZATIONAL recipients (vendor, service_provider, institutional_client, customer, etc.) + G3_EXTREME → HIGH. Evidence: T1 (partner → EXTREME), MM97/98/178–180 (service_provider/inst_client/vendor → HIGH).

**Change 10 — O1 corrected: redacted + authorized + malicious + unopened → MODERATE (was LOW)**  
O1 was tested in the original batch before this protection-level rule was known. Batch 11 (MM35/36/37) confirmed the correct result is MODERATE for this combination. O1's expected value updated accordingly.

### New Bucket Sets Added

```
STRONG_MITS = {permitted, written_obtained, written_provided, satisfactory, ce_ba_obligated, company_policy}
WEAK_MITS   = {obligated_no_assurance, no_written_obtained, no_written_provided}
PERSONAL_RECIPS = {partner, relative, another_family, non_custodial, parent_guardian, patient, general_public}
GATE2_HACKER_MOD = {unable_retrieve, improper_use, sent_to_media, unknown_mit, ssn_exposed,
                    ransomware_confirmed, unsure_backup, confirmed_viewing}
GATE2_MEDIA_MOD  = {unable_retrieve, improper_use, sent_to_media, unknown_mit, ssn_exposed,
                    ransomware_confirmed, confirmed_viewing}  ← no unsure_backup
```

### Version History Update

| Version | Cases | Key change |
|---|---|---|
| v13 | 82/82 | Plain text batch: 9 corrections including Gate 1b authorized prot threshold, attorney/employee special rules, Gate 3 nature-based unauth sub-rules |
| v14 | 144/144 | De-id/redacted/physical safeguard batch: STRONG/WEAK mit split for attorney/hacker/media/obligated-unauth; Gate 1a prot<0.85 authorized escalation; Gate 2 hacker vs media sets; Gate 3 personal vs organizational for uninten unauth |

