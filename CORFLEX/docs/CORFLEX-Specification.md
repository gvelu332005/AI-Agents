# CORFLEX — Legacy Container Specification

Source: ABSuite Matrix DB (`Matrix_FLEXINT` on `CR25ABSSQLBLD1`), container `CORFLEX`
(ABSId `57C7CCE6-B5D2-11DF-A5B2-0019B91D877F`, model `FLEX_CORFLEX.model`, stereotype `Report`).

Container description on file: "CoRoutine for FLEXDB Access".

Line numbers below are per-owner source line numbers (`dbo.LinesOfCode.[LineNo]`). Where a
source line is inside an inserted block the number jumps, because inserted code is not expanded
in the calling owner's lines.

## Purpose

`CORFLEX` is a coroutine that other systems call to read FLEX (billing) data. It has no screen
(`HasScreen = 0`) and no named parameters (`HasParameters = 0`). The caller exchanges data
through the shared `GLB.PARAM` block (see [Parameter contract](#parameter-contract)). There are 10
`Frame*` children, each with a `.Main`:

| Frame | Container description on file | Role |
|---|---|---|
| `Frame10` | ACL,APL and PE040 | Policy term selection and billing summary |
| `Frame11` | EFMST Read | EFT/RBP indicator (called by `Frame10`) |
| `Frame20` | MIS/PE020 and WARDSPAYL | Statistical record fields (first/last record) |
| `Frame30` | MIS Stat Info | Statistical record fields for the loss-date term |
| `Frame40` | ECASH-ACL Cntrl Nbr Request | ECASH control-number lookup |
| `Frame50` | FHRAS access (Hurr Assmt) | Hurricane-assessment description lookup |
| `Frame60` | Predictive Modeling | Predictive-model data |
| `Frame61` | FHIST last payment type | Payment-type / count helper (called by `Frame60`) |
| `Frame70` | Billing Inquiry - ACL | Billing-inquiry screen data |
| `Frame90` | debugging - print glb.param | Debug extractor (not called) |

## Parameter contract

Every request starts from `SG_PARAM_INFO` (2000 bytes), which `Main` loads with
`MV GLB.PARAM SG_PARAM_INFO` (line 7).

| Field | Type | Length | Direction | Meaning |
|---|---|---|---|---|
| `SD_PR_RQST` | AsNumber | 2 | In | Request code `01`-`07` |
| `SD_PR_ERRFLG` | AsString | 1 | Out | Error/status flag (see [Error flag values](#sd_pr_errflg-values)) |
| `SG_PR_COBRPO.SD_PR_COB` | AsNumber | 4 | In | Company/branch ("   Company Branch") |
| `SG_PR_COBRPO.SD_PR_PREFIX` | AsString | 3 | In | Policy prefix |
| `SG_PR_COBRPO.SD_PR_POLNO` | AsString | 8 | In | Policy number |
| `SD_PR_APFLG` | AsString | 1 | In | Application flag (`C` triggers the branch-remap retry in `Frame10`) |
| `SD_PR_EFF_DT` | AsNumber | 8 | In/Out | Effective date. Request `03`: input = claim loss date. Request `01`-`03`: set to `FBPOL.EFF_DATE` on success |
| `SD_PR_EXP_DT` | AsNumber | 8 | In/Out | Expiration date of the requested term. Request `03`: set to `FBPOL.EXP_DATE` on success |
| `SD_PR_DATA` | AsString | 1965 | Out | Response payload (`SG_PR_RQST`, 281 bytes, for requests `01`-`04`) |

Layout check: 2 + 1 + 15 + 1 + 8 + 8 + 1965 = 2000.

Requests `05`, `06` and `07` use different overlays of `GLB.PARAM` (`SG_FHRAS_INFO`,
`SG_PARAM_INFO` with `SD_PR_DATA` = `SG_PRDTCVMDL`, `SG_ACLBL_PARAM`); see the per-frame sections.

### `SD_PR_ERRFLG` values

Confirmed assignments. The business meaning in the last column is inferred.

| Where | Value | Condition | Inferred meaning |
|---|---|---|---|
| `Frame10` L18 | `"1"` | Request `03`: no term with expiration date >= loss date + 1 | Term not found |
| `Frame10` L52 | `"1"` | Requests `01`/`02`: requested term not found | Term not found |
| `Frame10` L60 | `"0"` | Same, and the latest term (`BACK`) is also not found | Policy not found |
| `Frame10` L49 | `"0"` | Request `03` with `GA_GS` = asterisk (appears unreachable, see note) | Policy not found |
| `Frame30` L7 | zeros | Initial value | Not found |
| `Frame30` L13 | `GC_01` | A `FBPOL` term exists for the policy | Policy found |
| `Frame30` L24 | spaces | Loss date in the term and a type-03 `FSTAT` record exists | Success |
| `Frame40` L6 | `"0"` | `ECASH` lookup fails | Not found |
| `Frame50` L8/L14 | `Y` then `N` | `Y` default; `N` when the `FHRAS` lookup succeeds | `Y` = error |

`Main` calls `Frame20`/`Frame30` only when `SD_PR_ERRFLG` = spaces after `Frame10`
(lines 16 and 22), which confirms spaces = success for `Frame10`.

## `CORFLEX.Main` — request dispatcher

ABSId `57C7CD48-B5D2-11DF-A5B2-0019B91D877F`, 51 source lines, 39 command rows.

```
SG_PARAM_INFO := GLB.PARAM                                  (L7)
INIT SG_PR_RQST                                             (L9)
BC SG_PARAM_INFO.SD_PR_RQST
  CS 01  Frame10                                            (L11-L12)
  CS 02  Frame10; if ERRFLG = spaces then Frame20           (L14-L18)
  CS 03  SD_AMEND_DT := SD_PR_EFF_DT; Frame10;
         if ERRFLG = spaces then Frame30                    (L19-L24)
  CS 04  Frame40                                            (L25-L26)
  CS 05  Frame50; JTO F00_ENDLG                             (L27-L29)
  CS 06  Frame60; JTO F00_ENDLG                             (L30-L32)
  CS 07  Frame70; JTO F00_ENDLG                             (L34-L36)
EC
SG_PARAM_INFO.SD_PR_DATA := SG_PR_RQST                      (L40)
GLB.PARAM := SG_PARAM_INFO                                  (L41)
LAB F00_ENDLG                                               (L51)
```

| Request | Comment on the `CS` line | Frames | Output path |
|---|---|---|---|
| `01` | `ACL,APL,PE040` | `Frame10` | `SG_PR_RQST` in `SD_PR_DATA`, `GLB.PARAM` by `Main` |
| `02` | `PE020,WARDSPAYL` | `Frame10`, `Frame20` | same |
| `03` | `STATS` | `Frame10`, `Frame30` | same |
| `04` | `ACL ECASH CNTRL#s` | `Frame40` | same |
| `05` | `ACTG/ASHURR` | `Frame50` | `Frame50` writes `GLB.PARAM` itself (L18) |
| `06` | `Predictive Modeling` | `Frame60` | `Frame60` writes `GLB.PARAM` itself (L151) |
| `07` | `Billing Inquiry in ACL` | `Frame70` | `Frame70` writes `GLB.PARAM` itself (L2921) |

Observations:

- There is no `OW` branch: an unknown request code runs no frame and returns the `INIT`-ed
  `SG_PR_RQST` in `SD_PR_DATA` with `SD_PR_ERRFLG` unchanged.
- Commented-out code: an `EX 90 AS ... (CORFLEXKAT)` call under request `01` (L13) and an email
  notification for request `03` (L43-L48, "I've turned this off for a bit-CL").

## Frame inventory

| Frame | Entities (confirmed from `DBAccess` rows) | Called by | Calls | Writes |
|---|---|---|---|---|
| `Main` | none | external | `Frame10`, `Frame20`, `Frame30`, `Frame40`, `Frame50`, `Frame60`, `Frame70` | none |
| `Frame10` | `FBPOL` (4 `Determine`), `ACTNM` (`LookUp`), `FCNCL` | `Main` | `Frame11` | none |
| `Frame11` | `EFMST` | `Frame10` | none | none |
| `Frame20` | `FSTAT` | `Main` | none | none |
| `Frame30` | `FBPOL`, `FSTAT` | `Main` | none | none |
| `Frame40` | `ECASH` (`LookUp`) | `Main` | none | none |
| `Frame50` | `FHRAS` (`LookUp`) | `Main` | none | none |
| `Frame60` | `FBPOL`, `EFMST`, `FHIST` (3 `Determine`) | `Main` | `Frame61`, `GP_CANCEL_REASONS` | none |
| `Frame61` | `PDHST` | `Frame60` | `GP_HISTORY_TYPE` | none |
| `Frame70` | `FBPOL`, `NOTFY`, `FCNCL`, `EFMST`, `FBSCN`, `FHIST`, `FDAYC`, `CMNTS`, `ECASH`, `EFACT`, `EFHST`, `PCASH`, `PNDCR` | `Main` | `GP_CORACL`, `GP_AMT_DISP`, `GP_CANCEL_REASONS`, `GP_HISTORY_TYPE`, 10 `GI_BLD_*` inserts | `FBSCN` (store, update, purge) |
| `Frame90` | none | none found | none | none |

`GP_CANCEL_REASONS` is called from `Frame60` (L97) and `Frame70` (L769) only. `Frame10` calls only
`Frame11` (its Katrina block is commented out).

## Deep-dive control flow

### `Frame10.Main` — policy term selection and billing summary (ABSId `57C7CD38-B5D2-11DF-A5B2-0019B91D877F`)

198 source lines, 123 command rows.

1. **Next-term date (L7):** `AD GC_01 SD_AMEND_DT GIV SD_AMEND_DT1`. Comment: "policies are not
   inforce on their expiration date so we need the following term - adding 1 to the expiration
   date". `SD_AMEND_DT1` = `SD_AMEND_DT` + 1.
2. **Term selection:**
   - Request `03` (L10-L21): `DT GROUP FBPOL.FBPOLP00 (COB, PREFIX, POLNO) FROM (SD_AMEND_DT1)`
     — first term at or after the loss date + 1. Not found (`GA_GS` = asterisk) -> `SD_PR_ERRFLG` =
     `"1"`, jump to `END_LG10`.
   - Other requests (L22-L45): `DT EVERY FBPOL.FBPOLP00 (COB, PREFIX, POLNO, SD_PR_EXP_DT)` —
     exact expiration-date match. If not found and the request is `01` with `SD_PR_APFLG` = `"C"`:
     take branch = COB positions 3-4; if it is between 30 and 39, subtract 30 from the COB
     (`SD_TMP_COB`) and retry with `DT GROUP ... BACK (GC_99999999)` (latest term).
3. **Not found (L47-L63):** request `03` -> `"0"` and exit (appears unreachable, because the
   request-`03` miss already exited at L18). Other requests -> `"1"`, then retry the latest term
   with `DT GROUP ... BACK`; if that also fails -> `"0"`. Exit in both cases.
4. **Returned dates (L65-L71):** `SG_STATUS_NSF` cleared; `SD_ACL_DATE` formatted from
   `SD_PR_EXP_DT` (`MM-DD-CCYY`). Request `03`: `SD_PR_EXP_DT` := `FBPOL.EXP_DATE`. All requests:
   `SD_PR_EFF_DT` := `FBPOL.EFF_DATE`.
5. **Response fields (L72-L114)** — see the mapping table below. `Frame11` is called at L88 for
   the EFT indicator.
6. **Alert/e-bill (L98-L110):** `LU FBPOL.ACCOUNT ACTNM`; on success `SD_PR_ALRTDYS`, `SD_PR_EBILL`,
   `SD_PR_EMLADDR` come from `ACTNM.ALERT_DYS`, `ELCTR_BILL`, `EMAIL_ADDR`; otherwise zeros/spaces.
   A blank e-bill flag becomes `N`.
7. **Category (L113-L114):** `SD_PR_CAT` := `FBPOL.CAT_ID`, then immediately overwritten with `"00"`,
   so the returned category is always `00`.
8. **Katrina block (L117-L132):** commented out; the source says it "needs to remain for future
   hurricane use" (commented out 12/2005).
9. **Status message (L135-L191)** — `BC FBPOL.POL_STATUS`:

| `POL_STATUS` | `SD_ACL_STATUS` | `SD_ACL_DATE` | `SD_PR_BLLSTAT` |
|---|---|---|---|
| `FL` | `POLICY IS LAPSED` | spaces | `L` |
| `CF`, `CP` | `POLICY IS CANCELLED` | spaces | `C` |
| `IC` | `INIT CANC` | `FBPOL.PCN_SND_DT` (`MM-DD-CCYY`) | space |
| `PC` | `POL WILL CANCEL ON` | `FBPOL.FNLCANC_DT` | `P` |
| `CC` | `POL SHOULD CANCEL ON` | `FBPOL.FNLCANC_DT`, or `99-99-9999` if it is `99999999`; replaced by `FCNCL.CANC_EFFDT` when an `FCNCL` record exists | `S` |
| other | `BILL DUE ON` if `BILLDUE_DT` <> `99999999` (date = `BILLDUE_DT`), else spaces | | unchanged |

   `SD_PR_BILLMSG` := `SG_STATUS_NSF` (with `NSF-` prefix) when `FBPOL.NSF_POLICY` = `Y`; for status
   `IC` it is the whole group; otherwise only `SG_ACL_ALL`.
10. **Totals (L193-L196):** `SR_PR_STATSH` := `FBPOL.STAT_RELSD`; `SD_PR_TERMPD` :=
    `FBPOL.TERM_PAID` + `FBPOL.PNDG_CRDT` (change tag OB-7084).

Response mapping (`SG_PR_RQST`):

| Output field | Source |
|---|---|
| `SD_PR_ACCOUNT` | `FBPOL.ACCOUNT` |
| `SD_PR_AGENT` | `FBPOL.AGENT_NBR` |
| `SD_PR_BILLDAY` | `FBPOL.BILLG_DAY` |
| `SD_PR_STATUS` | `FBPOL.POL_STATUS` |
| `SD_PR_TOTDUE` | `FBPOL.TOTAL_DUE` |
| `SD_PR_MINDUE` | `FBPOL.BILLAMTDUE` |
| `SD_PR_DUE_DT` | `FBPOL.BILLDUE_DT` |
| `SD_PR_PCN_DT` | `FBPOL.PCN_SND_DT` |
| `SD_PR_FNL_DT` | `FBPOL.FNLCANC_DT` |
| `SD_PR_LPS_DT` | `FBPOL.LAPSE_DATE` |
| `SD_PR_NSF_FLG` | `Y` if `FBPOL.NSF_POLICY` = `Y`, else `N` |
| `SD_PR_PCNLPS` | `FBPOL.PCN_NBR` |
| `SD_PR_EFT_FLG` | `GG_EFT_INFO.GA_EFT_FOUND` (from `Frame11`) |
| `SD_PR_PRMST` | `FBPOL.PREM_STATE` |
| `SD_PR_PMTRCVD` | `FBPOL.PYMTS_RCVD` |
| `SD_PR_INSNM` | `FBPOL.INS_NAME` |
| `SD_PR_MLGCITY` / `SD_PR_MLGST` | `FBPOL.CITY` / `FBPOL.STATE_ABBR` |
| `SD_PR_REIN_YN` | `N` (overridden by `Frame20`/`Frame30`) |
| `SD_PR_EFT_RBP` | `SD_EFT_RBP` (from `Frame11`) |
| `SD_PR_ALRTDYS`, `SD_PR_EBILL`, `SD_PR_EMLADDR` | `ACTNM` (see step 6) |
| `SD_PR_CAT` | always `00` |
| `SD_PR_BLLSTAT`, `SD_PR_BILLMSG` | status table above |
| `SR_PR_STATSH` | `FBPOL.STAT_RELSD` |
| `SD_PR_TERMPD` | `FBPOL.TERM_PAID` + `FBPOL.PNDG_CRDT` |

### `Frame11.Main` — EFT/RBP indicator (ABSId `57C7CD3C-B5D2-11DF-A5B2-0019B91D877F`)

1. `INIT GG_EFT_INFO` (L3).
2. Key (L5-L16): if `FBPOL.WHOPAY_IND` = `I` and `FBPOL.ACCOUNT` <> zeros -> `SD_REC_TYP` = `01`,
   key = account number; otherwise `SD_REC_TYP` = `02`, key = COB + prefix + policy (`GG_COBRPOL`).
3. `DT EVERY EFMST.EFMSTP01 (SD_REC_TYP, GA_KEYFLD2)` (L17).
4. Not found -> `GA_EFT_FOUND` = `N`. Found -> `Y`, `SD_EFT_RBP` = `E`, then:
   - `EFMST.EFT_PAC_ID` = `R` -> `GA_EFT_FOUND` = `R`, `SD_EFT_RBP` = `R`
   - `EFMST.HOLD_FLG` = `Y` -> `GA_EFT_FOUND` = `H`, `SD_EFT_RBP` = spaces (OSP-608)
   - `EFMST.BGN_DATE` > system date -> `N`
   - `EFMST.END_DATE` <= system date -> `N`, `SD_EFT_RBP` = spaces (OB-9911)

   The field comment on `SD_PR_EFT_FLG` says "Y-EFT, H-Held, N-No"; the code can also return `R`.

### `Frame20.Main` — statistical record fields (ABSId `57C7CD34-B5D2-11DF-A5B2-0019B91D877F`)

`DT GROUP FSTAT.FSTATP00 (FBPOL.COB, PREFIX, POLICY, EXP_DATE) FROM (zeros)` (L1). For each record
copy `FSTAT.STAT_IMAGE` into `GG_STAT_RECORD`; for record type `03` (L4) set:

| Output | Source |
|---|---|
| `SD_PR_TRANCD` | `GN_STAT_TRANCODE` |
| `SD_PR_MLCV` | `GA_STAT_MLCV` |
| `SD_PR_DECCD` | `GA_STAT_DEC_CODE` |
| `SD_PR_SUBLN` | `GN_STAT_SUBLINE` |
| `SD_PR_CLASSCD` | `GN_STAT_CLASS2` |
| `SD_PR_LOCNO` | `GA_STAT_LOC_UNIT` |
| `SD_PR_BLDNO` | `GA_STAT_BLDG` |
| `SD_PR_AMTINS` | `GN_STAT_AMT_INS` |
| `SD_PR_LOCZIP` | `GN_STAT_LOC_ZIP` |
| `SD_PR_DACIND` | `01`; if trancode < 070, its 2nd digit `2` -> `02`, `7` -> `03` |

Reinsurance (L24-L33): trancode `17`, `27` or `57` with `GN_STAT_GROSS` > 0 -> `SD_PR_REIN_YN` = `Y`,
`SD_PR_REINCO` = `GN_STAT_REIN_CO`, `SD_PR_REINSPC` = `GN_STAT_REIN_PCT`, and the loop exits (`BK`).

Observation: the loop only exits on the reinsurance case, so otherwise the **last** type-03
record's values remain. `SD_PR_DACIND` has the field comment "D-Direct,A-Assmd,C-Ceded" but is set
to the numbers `01`/`02`/`03`.

### `Frame30.Main` — statistical record for the loss-date term (ABSId `57C7CD30-B5D2-11DF-A5B2-0019B91D877F`)

Comment (L1-L5): "The field SD-PR-EFFDT (claims loss date) sets the SD-AMEND-DT".

1. `SD_PR_ERRFLG` := zeros (L7).
2. `DT GROUP FBPOL.FBPOLP00 (COB, PREFIX, POLNO) FROM (zeros)` over all terms (L9). For each term:
   `SD_PR_ERRFLG` := `GC_01` (L13); if `SD_AMEND_DT` < `EFF_DATE` or >= `EXP_DATE`, skip the term
   (`JTO END_DT03`, L14-L16).
3. Otherwise read `FSTAT.FSTATP00` for the term (L19) and apply the same type-03 copy and
   reinsurance logic as `Frame20` (L23-L53), additionally setting `SD_PR_ERRFLG` to spaces (L24).

### `Frame40.Main` — ECASH control-number request (ABSId `57C7CD2C-B5D2-11DF-A5B2-0019B91D877F`)

First line `Runtime_Context__ := "REPORT"; : Added by migration`.
`LU SG_PARAM_INFO.SG_PR_COBRPO ECASH` (L4) — the 15-byte `SG_PR_COBRPO` group value is used as the
`ECASH` key. Not found -> `SD_PR_ERRFLG` = `"0"`. Found -> `SD_PR_AGENT` := `ECASH.AGENT_NBR`,
`SD_PR_PRMST` := `ECASH.STATE_NBR`, `SD_PR_INSNM` := `ECASH.INS_NAME`.

### `Frame50.Main` — hurricane-assessment lookup (ABSId `A6A23A09-45A2-11E0-A64E-0019B91D877F`)

`SG_FHRAS_INFO := GLB.PARAM`; `SD_HA_ERRFLG` := `Y`; `SD_HA_DESC`, `SD_HA_FLR` := spaces;
`LU SD_HA_HANBR FHRAS`. Found -> `SD_HA_ERRFLG` := `N`, `SD_HA_DESC` := `FHRAS.HA_NAME1`.
`GLB.PARAM := SG_FHRAS_INFO`.

`SG_FHRAS_INFO` layout (2000): `SD_HA_RQST` (2, number), `SD_HA_ERRFLG` (1), `SD_HA_HANBR` (3, number,
input), `SD_HA_DESC` (21), `SD_HA_FLR` (1973 filler).

### `Frame60.Main` — predictive-model data (ABSId `F179749C-3DE6-4FDD-A8FF-94BFF7304D64`)

Output block `SG_PRDTCVMDL` (90 bytes), moved into `SG_PARAM_INFO.SD_PR_DATA` and returned in
`GLB.PARAM` (L150-L151).

1. **Policy lookup (L3-L12):** `DT GROUP FBPOL.FBPOLP03 (SD_PR_POLNO) BACK (GC_9999, GLB.HIGH,
   GC_99999999)` — latest term found by policy number (the alternate-key profile `P03`). Not found
   -> jump to `F11END` (L152) **without** writing `GLB.PARAM`.
2. **Counters reset (L14-L19):** `SD_PCN`, `SD_PYMTS`, `SD_TRNSFRM`, `SD_TRNSFTO`, `SD_TN`, `SD_REINS` := 0.
3. **Direct fields (L21-L29):** `SD_PM_COB`, `SD_PM_POLNO`, `SD_PM_EFFDT`, `SD_PM_EXPDT`, `SD_PM_STATE`
   (`PREM_STATE`), `SD_PM_INSNM` (`INS_NAME`), `SD_PM_NBRBR` (`NSF_NBR`), `SD_PM_NBRPMTSLT` (`PYMTS_LATE`).
4. **Installments `SD_PM_INSTL` (L32-L53):** `GN_TMP_TERM` := `TERMINMOS` clamped to 1-12.
   `GN_MAX_TERM` := 9 when COB positions 3-4 = `04`, else 12; if `GN_TMP_TERM` < `GN_MAX_TERM` then
   `GN_MAX_TERM` := `GN_TMP_TERM`. The value is therefore `MIN(clamped TERMINMOS, 9 or 12)`.
   It is forced to `01` when `WHOPAY_IND` = `M` or `BOND_FLG` = `Y`.
5. **EFT `SD_PM_EFT` (L56-L89):** key = account (rec type `01`, if `WHOPAY_IND` = `I` and an account
   exists) or policy (rec type `02`); `DT EVERY EFMST.EFMSTP01`. `Y` unless: not found, `BGN_DATE` >
   policy expiration, `END_DATE` <= policy expiration, held with `HOLD_DATE` <= expiration, or
   `WHOPAY_IND` <> `EFMST.DPSTR_TYP`. Different from `Frame11` because it compares to the
   **policy expiration** instead of the system date.
6. **Cancel reason `SD_PM_CNCREAS` (L93-L98):** if `FBPOL.CANC_REASN` <> zeros, call
   `GP_CANCEL_REASONS`; result `GA_MINIMSG` (14 bytes).
7. **Last payment type, transfers and counts (L102-L147):** scan `FHIST.FHISTP00` newest first
   for the term's expiration date and call `Frame61` per record (pass `SD_PM_PASS` = `01`). If a
   transfer-from policy was captured (`SG_TRNSF_FR`), scan that policy (pass `02`); if still no
   type, scan the current policy again (pass `03`); if still blank, `CHCK`.
8. **Counts (L113-L123):** `SD_PM_LSTPMTTYP` := `SD_PAYTYP`; `SD_PM_PCNSNT` := `FBPOL.PCN_NBR` -
   `SD_TN` (or `PCN_NBR` if the difference is negative; comment "FBUPD adds TN+PCN");
   `SD_PM_NBRPMTS` := `PYMTS_RCVD`; `SD_PM_NBRLPS` := `SD_TN`; `SD_PM_REINSSNT` := `SD_REINS`.
9. A trailing comment (L158) mentions `Frame62.Main() :pull policies`; no `Frame62` exists.

### `Frame61.Main` — payment-type and count helper (ABSId `578F17A5-BC27-444A-BC30-E8A064194570`)

Called once per `FHIST` row. `GG_FHIST_INFO.GN_ACT_TYPE` := `FHIST.ACT_TYP`; `GP_HISTORY_TYPE`
sets `GA_PAYMENT_FLG`.

- **Payment type (only if `GA_PAYMENT_FLG` = `Y` and `SD_PAYTYP` still blank):**

| `ACT_TYP` | `ACT_SUBTYP` | `SD_PAYTYP` |
|---|---|---|
| 08, 09, 10, 29, 30, 31 | 01 / 02 / 03 / other | `CC` / `MO` / `DBT` / `CHCK` |
| 42, 45, 51 | 01 / 02 / 03 / other | `CC` / `MO` / `DBT` / `CHCK` |
| 46 | any | `EFT`; becomes `ACH` if a `PDHST.PDHSTP02 (AGENT_NBR, ACT_DATE, first 15 of ACT_DESC)` row has `USERCODE` <> spaces ("other than monthly eft") |
| 62, 63 | 01 or 11 / 03 or 13 / 04 or 14 / other | `CC` / `DBT` / `ACH` / `ACH` |
| 64 | (pass 1 only) | captures `FHIST.ACT_DESC` into `SG_TRNSF_FR` |
| 67 | 01, 04, 05, 10, 12 (pass 1 only) | captures `FHIST.ACT_DESC` into `SG_TRNSF_FR` |

- **Counts (pass 1 only, `SD_PM_PASS` = `01`):**

| `ACT_TYP` | Counter |
|---|---|
| 15 (termination/lapse) | `SD_TN` + 1 |
| 16 (PCN) | `SD_PCN` + 1 |
| 37, 38, 39 (UW reinsurance) | `SD_REINS` + 1 |
| 66 (transfer from), subtypes other than 02, 03, 08, 11 | `SD_TRNSFRM` + 1 |
| 67 (transfer received), subtypes 02, 03, 08, 11 | `SD_TRNSFTO` + 1 |

First line `Runtime_Context__ := "REPORT"; : Added by migration`. The `SD_PYMTS` increment is commented out.

### `Frame70.Main` — Billing Inquiry (ABSId `0EFED436-E3D1-11EA-83FC-522819897106`)

Description "Billing Inquiry - ACL". 1,788 code rows (source lines run to about 2,920 because
inserted blocks are not expanded), 1,608 command rows. Output block `SG_ACLBL_PARAM` (5087 bytes),
returned with `MV SG_ACLBL_PARAM GLB.PARAM` at `END_LOG` (L2921).

#### Phase 1 — setup and policy (L1-L37)

1. Purge the caller's scratch rows: `FBSCN.FBSCNP01 (GLB.USERCODE, GLB.Station, "ACLBL")` -> `Purge` (L2-L7).
2. `GW_WORK` COB/prefix/policy := the request's policy; `GWA_POL_VS_ACCT` := `P` ("Always P"); `SG_ACLBL_PARAM` := spaces.
3. `DT GROUP FBPOL.FBPOLP00 (COB, PREFIX, POLNO) BACK (99999999)` — latest term. Not found ->
   `JUMPTO END_LOG` (CL-2396); the response is returned all spaces with no error flag.
4. `GP_CORACL` fetches ACL policy data into
   `GWG_ACL_PFILE` (L32). `GN_DATE` := system date (OB-6511).
5. `ACCTNBR` := `FBPOL.ACCOUNT`; `EFFDT` := `FBPOL.EFF_DATE`; `EXPDT` := `FBPOL.EXP_DATE`.

#### Phase 2 — mortgagee and payor (L39-L165)

- `GA_WHOPAY_TYP` := `B` if `FBPOL.BOND_FLG` = `Y`, else `M`.
- `NOTFY.NOTFYP00 (COB, PREFIX, POLICY, EXP_DATE, GA_WHOPAY_TYP) BACK` -> mortgagee block
  `SG_MORT.NAMEADDR1`..`5` (36 bytes each): name, name 2, address line 1, address line 2, then
  `CITY STATE_ABBR ZIPCODE`; for non-bond policies, a final line `Loan Number: <LOAN_ID>`.
- Payor `SG_PAY.NAMEADDR1`..`5` by `FBPOL.WHOPAY_IND`:
  - `O` -> `NOTFY` record of type `O` (same name/address layout, 4 lines, 5th line spaces)
  - `M` -> copy of the mortgagee block
  - other -> spaces (comment: "if A, leave blank & Agent info will fill it on the ACL side")

#### Phase 3 — status text and EFT (L168-L349)

- `GA_STATUS_INQ` by `FBPOL.POL_STATUS`:

| Status | Text |
|---|---|
| `PP` | `Pending Payment - ` + `Bill Due On <date> <amount>` or `No Bill Currently Due` |
| `II` | `Expired Term ` / `Renewal Term ` / `Inforce Policy ` (by system date) + ` - ` + bill text as above |
| `IC` | `Initial Stage Of Cancel - NonPayment` |
| `FL` | `Policy Term Final Lapsed` |
| `PC` | `Policy Cancel Notice/Bill Sent - Canc Effective <FNLCANC_DT>` |
| `CC` | `Final Cancel Process - Waiting Coding - Canc Effective <FCNCL.CANC_EFFDT or 99-99-9999>` |
| `CF` | `Cancelled Flat <PTD_POLICY> <CANC_PREM + REINS_PREM>` |
| `CP` | `Cancelled Mid-Term <PTD_POLICY> <CANC_PREM + REINS_PREM> E/P Due: <WO_PREM>` |
| other | spaces |

  Then, if the expiration date is before the system date and status is `II` with `TOTAL_DUE` = 0:
  `Expired - Nothing Due` (`WO_PREM` = 0) or `Expired - Earned Premium Due`.
- `EFT_INFO` (14 bytes) starts as `EFT`; `EFMST.EFMSTP01` is keyed like `Frame11` (L279-L349):
  - Not found -> spaces.
  - Found (not RBP): begin date in the future or end date <= today -> spaces; hold flag `Y` ->
    `EFT HELD`; otherwise `EFT` is kept.
  - Found with `EFMST.EFT_PAC_ID` = `R` (RBP): active -> `AUTO-` followed by 4 characters of
    `EFMST.BANK_ACCT` (positions 8-11); not active (begin in the future, ended or held) -> spaces,
    except when held -> `AUTO-` + the same 4 characters + `HELD`.
  See [sensitive fields](CORFLEX-Data-Sources.md#sensitive-fields).

#### Phase 4 — "Important Info" (L351-L1022)

`LOOP` for `GN_LBCNT` = 1..33, building `GG_IMPORTANT_INFO` (two columns, 87 bytes) and moving it to
`SG_ACLBL_PARAM.SG_IMPINFO.L1`..`L33`. Amount fields go through `GP_AMT_DISP` (sign/format into a
14-byte display). Line map:

| Line | Left label = value | Right label = value |
|---|---|---|
| L1 | `Policy Status: ` + `POL_STATUS` + `GA_STATUS_INQ` | (single field) |
| L2 | `# Pay Plan Installments` = max installments (same rule as `Frame60`, but max always 12) | `Paid-To Date` = `PTD_POLICY` |
| L3 | `# Bills Sent` = `BILLS_SENT` (capped at max) | `Payment Reversal` = spaces |
| L4 | `# Bills Left` = max - `BILLS_SENT` (0 if over max, `TERMINMOS` or 12) | `Payment Reversal Bal Due` = `ACTG_CHRG` |
| L5 | `# Payments Received` = `PYMTS_RCVD` | `Pending Credit` = `PNDG_CRDT` |
| L6 | `# Due Dates Missed` = `NBR_MISDUE` | `Pol# Previous` = `PRV_COB/PRV_PREFIX/PRV_POLNO` (prefix only if none) |
| L7 | `# Payments Late` = `PYMTS_LATE` | `Pol# Current` = COB/prefix/policy |
| L8 | `# Bank Returns i.e. NSF` = `NSF_NBR` | `Pol# Renewal` = `RNR_COB/PREFIX/POLNO` |
| L9 | `# PCNs/Terminations Sent` = `PCN_NBR` | `Prev Bill Date` = `LSTBL_DATE` (if not `99999999`) |
| L10 | `# Cancels UW-Coded @ NonPay` = 0 (fixed) | `Prev Billed Balance` = `LSTBL_TOTL` |
| L11 | `# Cancels Flat` = 0 (fixed) | `Prev Bill Endorsements` = `PRV_BLDEND` |
| L12 | `# Cancels Mid Term` = 0 (fixed) | `Refund Checks` = 0 (fixed) |
| L13 | `Acct# Previous` = `ACCOUNT` (leading zeros stripped) | `Reinstate` = `REINS_PREM` |
| L14 | `Acct# Current` = `ACCOUNT` | `Remaining Term Balance` = `TOTAL_DUE` |
| L15 | `Acct# Renewal` = `ACCOUNT` | `Rollover` = `ROLLOVER` |
| L16 | `Audits` = 0 (fixed) | `S/C Installment Fees due` = `SRVCHG_DUE` |
| L17 | `Bank Return Amount` = 0 (fixed) | `S/C Installment Fees paid` = `SRVCHG_PD` |
| L18 | `Bank Return Bal Due` = `NSF_PYMTS` | `S/C Bank Return Fees due` = `FEE_DUE_BR` |
| L19 | `Billing Day` = `BILLG_DAY` | `S/C Bank Return Fees paid` = `FEE_PD_BR` |
| L20 | `Cancel Amt Unearned Prem` = `CANC_PREM` | `S/C Late Fees due` = `FEE_DUE_LT` |
| L21 | `Cancel UW-Coded for NonPay` = 0 (fixed) | `S/C Late Fees paid` = `FEE_PD_LT` |
| L22 | `Cancel Eff/PCN Due Date` = `FNLCANC_DT` (if not `99999999`) | `Standard Min Due Calc` = `BILLAMTDUE` |
| L23 | `Cancel Reason` = `GP_CANCEL_REASONS(CANC_REASN)` | `State Code/Premium State` = `PREM_STATE` |
| L24 | `Commission Paid Up Front` = `COMMUPFRNT` | `System Origin` = `BD-` + `SYSORIGIN` (bond) or `PERS_COMM` + `L-` + `SYSORIGIN` |
| L25 | `Dividends` = 0 (fixed) | `Term in Months` = `TERMINMOS` |
| L26 | `Due Date` = `BILLDUE_DT` | `Total Balance Due w/fees` = `TOTAL_DUE` + `FEE_DUE_BR` + `FEE_DUE_LT` |
| L27 | `Earned Premium Due` = `WO_PREM` | `Total Endorsements` = `TOTAL_ENDS` |
| L28 | `EFT Insured Setup Date` (or `EFT Begins` / `EFT Ended` / `HELD`) | `Total Initial Term Premium` = `NWRNR_PREM` |
| L29 | `EFT Held Date` = spaces | `Total Prem w/end,canc,etc` = `TERM_TOTAL` |
| L30 | `Fees,AutThft,MineSub,Tax,etc` = 0 (fixed) | `Total Term Paid` = `TERM_PAID` |
| L31 | `Final Lapse` = `LAPSE_DATE` | `Work Comp on policy` = `WCCODED` |
| L32 | `Last Entity Billed` = `LSTBL_ENTY` | `Write-off Billed` = `Y` if `WRITEOFF` = `01` |
| L33 | `Billing Center Account` = `GWA_PFILE_PCACCT` (from `GP_CORACL`) or `GWPOL.GW_ACCOUNT` | `Billing Center Policy` = `GWPOL.GW_PREFIX` + `GW_POLICY` (COB omitted; only when `GA_GW_POL` = `Y`) |

Hard-coded zeros: lines L10, L11, L12, L16 (left), L17 (left), L21, L25 (left), L30 (left) are
literal zero/blank values in the source, not data-driven.

#### Phase 5 — policy chain and scratch rows (L1031-L2455)

1. Purge `FBSCN` again (L1036-L1040), zero the accumulators in `GW_WORK`.
2. Walk the policy chain: start at the requested term (`FBPOLP00`, `SD_PR_EXP_DT`), then follow
   `FBPOL.PRV_COB/PRV_PREFIX/PRV_POLNO/PRV_EXP_DT` to earlier terms until there is no previous
   policy number or the term is not found (`LOOP`, L1117-L2455). The renewal-chain variant
   (`RNR_*`) is commented out.
3. For each term, ensure an `FBSCN` row of file `FBPOL` exists (`Store`, L1146-L1173), then add rows
   through inserted blocks:

| Source file / profile | Inserted block |
|---|---|
| `FHIST.FHISTP00` (term's expiration date) | `GI_BLD_HST_INQ` |
| `FDAYC.FDAYCP00 (01, A, ACCOUNT)`; `FDAYC.FDAYCP02 (POLICY, 01, P)` (rows with `MAINT` <> `D`) | `GI_BLD_FDAYC_CMNT` |
| `CMNTS.CMNTSP00 (ACCOUNT)` (policy blank, `MAINT` <> `D`); `CMNTS.CMNTSP02 (COB, PREFIX, POLICY)` | `GI_BLD_CMNT_INQ` |
| `ECASH.ECASHP02 (ACCOUNT)`; `ECASH.ECASHP03 (COB, PREFIX, POLICY)` | `GI_BLD_ECASH_CMNT` |
| `EFACT.EFACTP01 (04, COB/PREFIX/POLICY)` | `GI_BLD_PNDG_EFACT` |
| `EFHST.EFHSTP01 (04, COB/PREFIX/POLICY)` (posting date >= `GWN_FROMDT`) | `GI_BLD_PNDG_EFHST` |
| `ECASH.ECASHP02 (ACCOUNT)`, `ECASHP03 (policy)` | `GI_BLD_PNDG_ECASH` |
| `PCASH.PCASHP02 (ACCOUNT)`, `PCASHP03 (policy)` | `GI_BLD_PNDG_PCASH` |
| `ECASH.ECASHP04 (ACCOUNT)`, `ECASHP05 (POLICY)` where `CASH_SUBTY` <> `04` | `GI_BLD_ECASH_TRFR` |
| `PNDCR.PNDCRP01 (ACCOUNT)`, `PNDCRP02 (POLICY)` | `GI_BLD_PNDCR_INQ` |

   Each traced block (`GI_BLD_HST_INQ`, `GI_BLD_PNDCR_INQ`) searches `FBSCN.FBSCNP01` for an
   existing row with the same key and, if none, stores one `FBSCN` row (`FILENM` = source file,
   `REC_TYP`/`REC_SUBTYP`/`SEQ_NBR` from the source record, `AMOUNT` where applicable,
   `USERCODE`/`INITSTN`/`ISPECNM` from the caller) and counts it in `GN_FHIST_CNT`
   (history rows only).
4. `FBSCN` is keyed by user code, station, ispec (`"ACLBL"`), file name, policy-vs-account (`P`),
   key field, ... — it is a per-session scratch table.

#### Phase 6 — policy list (L2458-L2625)

`DT GROUP FBSCN.FBSCNP02 (GLB.USERCODE, GLB.Station, GA_ISPEC, "FBPOL", "P", key)` (`GA_ISPEC_LBNM`
is set to `LBPOLICIES`) reads the stored policy rows and, for each `FBPOL` term, builds
`GG_POL_TXT` (86 bytes): key
(COB/prefix/policy/exp), effective and expiration dates (`MM-DD-YY`), status, who-pays, billing day,
due date, total due (`TOTAL_DUE` + `FEE_DUE_BR` + `FEE_DUE_LT`, with `*` at position 5 of the key if
the total differs from `TOTAL_DUE`), earned-premium due, minimum due. The first 5 rows go to
`SG_POLICIES.L1`..`L5`; the totals row goes to the first empty line (L6 when all 5 are used).

- A policy with status `FL` that lapsed more than 182 days ago (`LAPSE_DATE` vs system date)
  is listed but **not added to the totals** (`JTO GDF_ARND`, L2537).
- Totals label: `  Policy Totals *Incl S/C Fees` when NSF or late fees are included, else `  Policy Totals`.
- `GN_CMNT_DATE` (set by the comment insert blocks) is returned in `CMNT_DATE` as `MM-DD-CCYY`.

#### Phase 7 — billing history (L2697-L2912)

Re-reads the `FBSCN` rows of file `FHIST` oldest to newest (comment "Always going to be Oldest to
Newest"). For each row, reads `FHIST.FHISTP00` and builds `SG_HIST_TXT` (86 bytes):

- Key `SG_HIST_KEY` (COB/prefix/policy, activity date, sequence) moved to `SG_LBHIST.SD_LBHKEY`.
- Activity date (`MM-DD-YY`), description from `GP_HISTORY_TYPE` (`GA_ACT_DESC`, 22 bytes), effective-due
  date (blanked for payment-like types), amount (`ACT_AMT1`, sign reversed via `Contra()` for
  payment/credit activity types), source/min amount (`ACT_AMT2` for most types).
- Running balance (`GWN_BALANCE`) accumulated by activity-type group (positive adds, payments subtract
  using `ACT_AMT1 - ACT_AMT2 - ACT_AMT4 - ACT_AMT5`), written to `SD_TXTH_BALANCE` and back to
  `FBSCN.AMOUNT` (`FL`).
- An `*` is placed in position 9 for types 16, 24, 40, 52, and for type 54 with a subtype, when
  `ACT_AMT4` or `ACT_AMT5` is non-zero.
- Only the **latest 15 rows** are returned: `SD_FHIST_CNT` = `GN_FHIST_CNT` - 15 when the count
  exceeds 15, and a row is kept when `GN_FHIST_CNT` <= 15 or its sequence is above that threshold.
  They go to `SG_ACLBL_PARAM.SG_INFO.L1`..`L15` (86 bytes each).

#### `SG_ACLBL_PARAM` layout (5087 bytes)

| Field | Length | Content |
|---|---|---|
| `SG_MORT.NAMEADDR1`-`5` | 5 x 36 | Mortgagee/bond obligee name and address |
| `SG_PAY.NAMEADDR1`-`5` | 5 x 36 | Payor name and address |
| `SG_IMPINFO.L1`-`L33` | 33 x 87 | "Important Info" lines |
| `SG_INFO.L1`-`L15` | 15 x 86 | Billing history lines |
| `SG_POLICIES.L1`-`L6` | 6 x 86 | Policy list (5) and totals |
| `CMNT_DATE` | 10 | Comment date |
| `ACCTNBR` | 10 | Account number |
| `EFFDT` / `EXPDT` | 8 / 8 | Term dates |
| `EFT_INFO` | 14 | EFT text |

Layout check: 180 + 180 + 2871 + 1290 + 516 + 10 + 10 + 8 + 8 + 14 = 5087.

### `Frame90.Main` — debug extractor (ABSId `57C7CD46-B5D2-11DF-A5B2-0019B91D877F`)

Description "debugging - print glb.param". Three lines:
`SD_PR_DATA := SG_PR_RQST`; `F90_DATA1 := SG_PARAM_INFO[1..129]`; `F90_DATA2 := SG_PARAM_INFO[130..250]`.
`F90_DATA1`/`F90_DATA2`/`F90_DATA3` are `Out` items of 130 bytes; `F90_DATA3` is never assigned. No
caller found (the only reference is the commented `EX 90` in `Main` L13).

## Container-level working storage

All private items except where noted. `SG_*` are groups.

| Item | Type / length | Purpose |
|---|---|---|
| `SD_AMEND_DT`, `SD_AMEND_DT1` | number 8 | Loss date and loss date + 1 (`Frame10`, request `03`) |
| `SD_EXP_DT` | number 8 | Expiration-date scratch |
| `SD_TMP_COB`, `SD_TMP_BR` | number 4 / 2 | COB and branch scratch (`Frame10`) |
| `SD_TRNCD3` | number 1 | 2nd digit of the statistical transaction code |
| `SD_REC_TYP` | number 2 | `EFMST` record type (`01` account, `02` policy) |
| `SD_EFT_RBP` | string 1 | `E`/`R`/space result of `Frame11` |
| `SD_PM_PASS` | number 1 | `Frame60`/`Frame61` pass (1, 2, 3) |
| `SD_PAYTYP` | string 4 | Last payment type (`CC`, `MO`, `DBT`, `CHCK`, `EFT`, `ACH`) |
| `SD_PCN`, `SD_PYMTS`, `SD_TN`, `SD_REINS`, `SD_TRNSFRM`, `SD_TRNSFTO` | number 2 | History counters (`Frame61`) |
| `SD_TMP_CNT` | signed 3 | PCN count scratch |
| `SV_CTRL` | string 15 | `PDHST` search control |
| `SD_FHIST_CNT`, `SD_REC_CNT`, `SD_CNTR` | number 8 | Billing history row counters (`Frame70`) |
| `SD_ACT_DESC`, `SD_CMNT`, `SD_FNLCNCDT` | string 15 / 78 / 10 | Description, comment and final-cancel-date scratch (not referenced by the traced code) |
| `SG_ACCOUNT.SD_ACCOUNT` | number 10 | Account key group |
| `SG_COBRPO` | group 15 | COB/prefix/policy group (`SD_COB`, `SD_PREF`, `SD_POLNO`) |
| `SG_TRNSF_FR` | group 55 | Transfer-from policy capture (`SD_TF_PVSA`, `SG_TF_GRP`, `SD_TF_EXPDT`, `SD_TF_CTL`, `SD_TF_USER`) |
| `SG_HIST_KEY`, `SG_HIST_TXT`, `SG_LBHIST` | 26 / 86 / 157 | Billing-history row structures |

`SG_HIST_TXT` layout (86): `SD_TXTH_ACT_DT` (8), `SD_TXTH_FLR1` (2), `SD_TXTH_ACTDESC` (22),
`SD_TXTH_FLR2` (1), `SD_TXTH_EFFDUE` (8), `SD_TXTH_FLR3` (1), `SD_TXTH_ACT_AMT` (14), `SD_TXTH_FLR4` (1),
`SD_TXTH_MIN_SRC` (14), `SG_TXTH_CBP` (`SD_TXTH_FLR5` 1, `SD_TXTH_BALANCE` 14).

`SG_PRDTCVMDL` layout (90): `SD_PM_COB` (4), `SD_PM_POLNO` (8), `SD_PM_EFFDT` (8), `SD_PM_EXPDT` (8),
`SD_PM_STATE` (2), `SD_PM_EFT` (1), `SD_PM_INSTL` (2), `SD_PM_NBRBR` (2), `SD_PM_NBRPMTS` (2),
`SD_PM_NBRPMTSLT` (2), `SD_PM_NBRLPS` (2), `SD_PM_PCNSNT` (2), `SD_PM_REINSSNT` (2),
`SD_PM_LSTPMTTYP` (4), `SD_PM_INSNM` (27), `SD_PM_CNCREAS` (14).

`SG_PR_RQST` layout (281) with field comments on file:

| Field | Length | Comment on file |
|---|---|---|
| `SD_PR_ACCOUNT` | 10 | Policy Account |
| `SD_PR_AGENT` | 6 | Agent |
| `SD_PR_BILLDAY` | 2 | Billing Day |
| `SD_PR_BLLSTAT` | 1 | ACL Billing Status |
| `SD_PR_BILLMSG` | 35 | Status Message |
| `SD_PR_STATUS` | 2 | Policy Status |
| `SD_PR_TOTDUE` / `SD_PR_TOTSGN` | 10 (2 dec) / 1 | Total Due / sign |
| `SD_PR_MINDUE` / `SD_PR_MINSGN` | 10 (2 dec) / 1 | Minimum Due / sign |
| `SD_PR_DUE_DT` | 8 | Bill Due Date |
| `SD_PR_PCN_DT` | 8 | PCN Date |
| `SD_PR_FNL_DT` | 8 | Final Cancel Date |
| `SD_PR_LPS_DT` | 8 | Lapse Date |
| `SD_PR_NSF_FLG` | 1 | Bank Returns |
| `SD_PR_PCNLPS` | 2 | PCN/Termination count |
| `SD_PR_EFT_FLG` | 1 | Y-EFT, H-Held, N-No |
| `SD_PR_PRMST` | 2 | Premium State Code |
| `SD_PR_PMTRCVD` | 2 | Payments Received |
| `SD_PR_INSNM` | 27 | Insured Name |
| `SD_PR_MLGCITY` / `SD_PR_MLGST` | 20 / 2 | Mailing City / State Abbr |
| `SD_PR_DACIND` | 1 | D-Direct, A-Assmd, C-Ceded |
| `SD_PR_REIN_YN` | 1 | Reinsurance Yes/No |
| `SD_PR_REINCO` | 3 | Reinsurance Company |
| `SD_PR_REINSPC` | 4 (3 dec) | Reinsurance Percent |
| `SD_PR_TRANCD` | 3 | Transaction Code |
| `SD_PR_MLCV` | 4 | Major Line and Coverage |
| `SD_PR_DECCD` | 2 | Dec Code |
| `SD_PR_SUBLN` | 4 | SubLine |
| `SD_PR_CLASSCD` | 6 | Class Code |
| `SD_PR_LOCNO` | 3 | Location-Unit |
| `SD_PR_BLDNO` | 3 | Building Number |
| `SD_PR_AMTINS` | 5 (signed) | Amount of Insurance |
| `SD_PR_LOCZIP` | 9 | Location Zip |
| `SD_PR_CAT` | 2 | Cat # |
| `SR_PR_STATSH` | 1 | Stats are Held in Flex Y/N |
| `SD_PR_EFT_RBP` | 1 | E-EFT, R-RBP, space |
| `SD_PR_ALRTDYS` | 2 | Alert Days 00, 05 |
| `SD_PR_EBILL` | 1 | EBILL N-No, Y-Yes |
| `SD_PR_EMLADDR` | 50 | Email Address |
| `SD_PR_TERMPD` | 9 (2 dec) | (no comment) |

`SD_PR_TOTSGN` and `SD_PR_MINSGN` are defined but not assigned in the traced code.

## Shared constants used

Lettered constants below were confirmed via `DataItems.InitialValue` in `Matrix_FLEXINT`; each equals
its suffix. Numeric constants (`GC_01`, `GC_02`, ...) were not queried and are assumed to equal their number.

| Constants | Values |
|---|---|
| `GC_01`-`GC_33` etc. | numeric request/type/loop codes (assumed) |
| `GC_Y`, `GC_N`, `GC_R`, `GC_H`, `GC_E`, `GC_P`, `GC_I`, `GC_O`, `GC_M`, `GC_B`, `GC_D` | `Y`, `N`, `R`, `H`, `E`, `P`, `I`, `O`, `M`, `B`, `D` |
| `GC_FL`, `GC_CF`, `GC_CP`, `GC_IC`, `GC_PC`, `GC_CC`, `GC_PP`, `GC_II` | `FL`, `CF`, `CP`, `IC`, `PC`, `CC`, `PP`, `II` (policy statuses) |
| `GC_99999999` | `99999999` (no-date sentinel) |
| `GC_ASTERISK` | `*****` (status after a failed lookup/determine) |
| `GC_FBPOL`, `GC_FHIST` | `FBPOL`, `FHIST` (file names stored in `FBSCN.FILENM`) |

## Environment and invocation

- `GP_CORFLEX` (L1-L25) is entirely commented out. Its logic: build `GSD-FLEX-PARAM`, then
  `GLB.SELFHOST = UFCPROD` -> `CALL COR (FLEX/CORFLEX ON BILLPK) (UFC)`; `UFCDEV` ->
  `CALL COR (TFLEX/CORFLEX ON BILLPKDEV) (ACT)`.
- `COROUTINE_CALL` L79: `:CS "CORFLEX" :don't see this being called in code` (commented out).
- `TESTFLEX.Frame11.Main` L10/L13: commented `Coroutine_FLEX_CORFLEX_BILLPK_UFC_Lib.report(GLB.PARAM)` and
  `Coroutine_TFLEX_CORFLEX_BILLPKDEV_ACT_Lib.report(GLB.PARAM)`.
- The coroutine libraries are `Coroutine_*_CORFLEX_*_Lib` containers (7), each with a `.report` child
  (the container ABSIds are in `corflex_container_info.txt`).

## Rewrite notes

- The single envelope + request code design maps cleanly to a request/response API with one
  operation per request code (`01`-`07`).
- Frames `10`/`20`/`30` all return the same `SG_PR_RQST` shape, composed in two steps; the
  composition (billing summary first, statistics only when `SD_PR_ERRFLG` = spaces) must be preserved.
- `Frame70` has a write side effect (per-user `FBSCN` scratch rows). A rewrite can build the
  response in memory and does not need the table, but it must keep the 15-row history cap, the
  5-policy cap plus totals row, and the 182-day lapsed exclusion from totals.
- Preserve the literal quirks: `SD_PR_CAT` always `00`; hard-coded zero lines in "Important Info";
  `SD_PR_DACIND` numeric codes; `SD_PR_EFT_FLG` can be `R`.

## Open questions

See [ReadMe.md — Open Questions](../ReadMe.md#open-questions).
