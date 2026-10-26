# CORFLEX Requirements

## Overview

CORFLEX is the FLEX-side coroutine (report) that gives other systems read access to the FLEX
billing database. The `CORFLEX` container in `Matrix_FLEXINT` is a **Report** stereotype
container with the description `CoRoutine for FLEXDB Access` (model file `FLEX_CORFLEX.model`,
owner container `FLEX`). It lives in the Matrix folders `FBPROD` ("Flex Production Rpts"),
`Reports` and `REPORTS_ONLINE` ("Reports for the Onlline").

CORFLEX has no screen and no named input/output parameters. A caller passes one 2000-byte
parameter block through `GLB.PARAM`; `CORFLEX.Main` copies it into `SG_PARAM_INFO`, dispatches on
the request code `SG_PARAM_INFO.SD_PR_RQST`, and returns the result in the same block (or, for
the larger responses, in a request-specific block moved back to `GLB.PARAM`). See
[docs/CORFLEX-Specification.md — `CORFLEX.Main`](docs/CORFLEX-Specification.md#corflexmain--request-dispatcher).

The container has 10 `Frame*` children, each with a `.Main` program. `CORFLEX.Main` dispatches
to 9 of them; `Frame11` (EFT lookup) is called by `Frame10`, `Frame61` (last-payment-type
helper) is called by `Frame60`, and `Frame90` is not called by anything in `Matrix_FLEXINT`.

CORFLEX is read-mostly. The only persistent writes are the `FBSCN` ("Screen Info") scratch rows
that `Frame70` (Billing Inquiry) builds and purges per user/station; no business file is
updated (see [docs/CORFLEX-Data-Sources.md — `FBSCN`](docs/CORFLEX-Data-Sources.md#fbscn--screen-info-scratch-file)).

## Scope

- Only use FLEX code and database structures that pertain to `CORFLEX` and its `Frame*.Main`
  children.
- Include the logic inserted directly into the container and the shared `GP_*` / `GI_BLD_*`
  routines it calls, only to the depth needed to describe CORFLEX's own behavior.
- Do not invent unconfirmed behavior. Items that were not traced are recorded under
  [Open Questions](#open-questions).
- Source of truth: `Matrix_FLEXINT` on `CR25ABSSQLBLD1` (read-only queries).

## Requirements

- Capture the CORFLEX purpose, request contract and dispatch table.
- Document each frame's confirmed behavior and the FLEX files it touches.
- Document the response layouts (`SG_PR_RQST`, `SG_PRDTCVMDL`, `SG_FHRAS_INFO`,
  `SG_ACLBL_PARAM`).
- Keep the structure as Overview / Scope / Requirements / Confirmed Frame Logic / Open Questions,
  with detail moved to `docs/`.

## Request Codes

`SG_PARAM_INFO.SD_PR_RQST` (2-digit number) selects the behavior. Full detail is in
[docs/CORFLEX-Specification.md](docs/CORFLEX-Specification.md#corflexmain--request-dispatcher).

| `SD_PR_RQST` | Source comment | Frames called | Returns |
|---|---|---|---|
| `01` | ACL, APL, PE040 | `Frame10` | Policy billing summary (`SG_PR_RQST`) |
| `02` | PE020, WARDSPAYL | `Frame10`, then `Frame20` if `SD_PR_ERRFLG` is spaces | Billing summary plus first statistical record fields |
| `03` | STATS | `Frame10`, then `Frame30` if `SD_PR_ERRFLG` is spaces | Billing summary plus statistical record for the term containing the loss date |
| `04` | ACL ECASH CNTRL#s | `Frame40` | Agent, premium state and insured name for an ECASH control number |
| `05` | ACTG/ASHURR | `Frame50` | Hurricane-assessment description (`FHRAS`) |
| `06` | Predictive Modeling | `Frame60` (calls `Frame61`) | Predictive-model record (`SG_PRDTCVMDL`) |
| `07` | Billing Inquiry in ACL | `Frame70` | Billing-inquiry screen data (`SG_ACLBL_PARAM`) |

## Confirmed Frame Logic

The following frame-level behavior is confirmed from the LDL+ source and `dbo.Commands` rows in
`Matrix_FLEXINT`. Line numbers are the per-owner source line numbers.

- `Main` ("F R A M E 0 0") copies `GLB.PARAM` into `SG_PARAM_INFO`, runs `INIT SG_PR_RQST`, and
  cascades on `SD_PR_RQST` (7 cases). Requests `01`-`04` finish by moving `SG_PR_RQST` into
  `SG_PARAM_INFO.SD_PR_DATA` and `SG_PARAM_INFO` into `GLB.PARAM`; requests `05`-`07` jump to
  `F00_ENDLG` and the frame itself writes `GLB.PARAM`.
- `Frame10.Main` (description "ACL,APL and PE040") touches `FBPOL`, `FCNCL` and `ACTNM`, and calls
  `Frame11`. It selects the policy term, then builds the billing summary
  (account, agent, billing day, status, amounts due, PCN/final-cancel/lapse dates, NSF flag, EFT
  flag, premium state, payments received, insured name, mailing city/state, alert days, e-bill
  flag and email address) and a status message.
- `Frame11.Main` (description "EFMST Read") touches `EFMST`: EFT/RBP indicator for the policy or
  the insured's account.
- `Frame20.Main` (description "MIS/PE020 and WARDSPAYL") touches `FSTAT`: copies the type-03
  statistical record fields and the reinsurance indicator.
- `Frame30.Main` (description "MIS Stat Info") touches `FBPOL` and `FSTAT`: same statistical
  fields as `Frame20`, but for the term whose effective/expiration dates contain the supplied loss
  date.
- `Frame40.Main` (description "ECASH-ACL Cntrl Nbr Request") touches `ECASH`: one lookup.
- `Frame50.Main` (description "FHRAS access (Hurr Assmt)") touches `FHRAS`: one lookup.
- `Frame60.Main` (description "Predictive Modeling") touches `FBPOL`, `EFMST`, `FHIST`; calls
  `Frame61` and `GP_CANCEL_REASONS`.
- `Frame61.Main` (description "FHIST last payment type") touches `PDHST`; calls `GP_HISTORY_TYPE`.
- `Frame70.Main` (description "Billing Inquiry - ACL") is the largest frame (about 2,920 source
  lines including inserted blocks). It touches `FBPOL`, `NOTFY`, `FCNCL`, `EFMST`, `FHIST`,
  `FBSCN`, `FDAYC`, `CMNTS`, `ECASH`, `PCASH`, `EFACT`, `EFHST`, `PNDCR`, calls `GP_CORACL`,
  `GP_AMT_DISP`, `GP_CANCEL_REASONS`, `GP_HISTORY_TYPE`, and inserts 10 `GI_BLD_*` blocks.
- `Frame90.Main` (description "debugging - print glb.param") has 3 lines of code and no database
  access. It is not called by `CORFLEX.Main` or any other owner found in `Matrix_FLEXINT`.

Field-level detail for each frame, the `Frame70` "Important Info" line map, and the billing
history logic live under [docs/](docs/):
[CORFLEX-Specification.md](docs/CORFLEX-Specification.md) and
[CORFLEX-Data-Sources.md](docs/CORFLEX-Data-Sources.md).

## Container-Level Working Storage

The container has 248 owners in `dbo.OwnerNames`: 226 data items and 22 containers. Container
data items are `Private` working storage; the `Public` items are the request/response groups
below.

| Group | Size | Purpose |
|---|---|---|
| `SG_PARAM_INFO` | 2000 | Request/response envelope: `SD_PR_RQST`, `SD_PR_ERRFLG`, `SG_PR_COBRPO` (COB 4 / prefix 3 / policy 8), `SD_PR_APFLG`, `SD_PR_EFF_DT`, `SD_PR_EXP_DT`, `SD_PR_DATA` (1965) |
| `SG_PR_RQST` | 281 | Billing-summary response used by requests `01`-`04` |
| `SG_PRDTCVMDL` | 90 | Predictive-model response for request `06` |
| `SG_FHRAS_INFO` | 2000 | Hurricane-assessment request/response for request `05` |
| `SG_ACLBL_PARAM` | 5087 | Billing-inquiry response for request `07` |
| `SG_STATUS_NSF` | 35 | Status message builder (`SD_ACL_NSF`, `SD_ACL_STATUS`, `SD_ACL_DATE`) |
| `SG_HIST_TXT` / `SG_HIST_KEY` / `SG_LBHIST` | 86 / 26 / 157 | Billing-history row builders for `Frame70` |
| `SG_TRNSF_FR` | 55 | Transfer-from policy captured by `Frame61` |

Full field layouts are in [docs/CORFLEX-Specification.md](docs/CORFLEX-Specification.md#container-level-working-storage).

## Legacy System Reference

Environment-specific coroutine libraries (all in folder `Coroutines`, each with a `.report` child):

| Library | Environment hint |
|---|---|
| `Coroutine_FLEX_CORFLEX_BILLPK_UFC_Lib` | Production (`FLEX`/`BILLPK`/`UFC`) |
| `Coroutine_TFLEX_CORFLEX_BILLPKDEV_ACT_Lib` | Development (`TFLEX`/`BILLPKDEV`/`ACT`) |
| `Coroutine_FLEXDEV8_CORFLEX_E8PACK_E8USER_Lib` | Dev 8 |
| `Coroutine_FLEXNEWQA_CORFLEX_NEWQAPACK_NEWQA_Lib` | New QA |
| `Coroutine_FLEXSIT_CORFLEX_SITPACK_SIT_Lib` | SIT |
| `Coroutine_FLXQA_CORFLEX_QAPACK_QA_Lib` | QA |
| `Coroutine_FLXTRAIN_CORFLEX_TRAINPACK_TRAIN_Lib` | Training |

`GP_CORFLEX` ("aka FlexLib/WebFBPLib", folder `GLB_LOGIC`) is the FLEX-side wrapper that used to
call the coroutine (`CALL COR (FLEX/CORFLEX ON BILLPK) (UFC)` in production,
`CALL COR (TFLEX/CORFLEX ON BILLPKDEV) (ACT)` in dev, selected on `GLB.SELFHOST`). Its whole body
is commented out.

Related: `GP_CORACL` is the FLEX-side routine `Frame70`
calls to fetch ACL policy data.

## Open Questions

- **Callers:** no active caller of CORFLEX was found in `Matrix_FLEXINT`. The only references are
  commented out (`GP_CORFLEX` L19/L22; `TESTFLEX.Frame11.Main` L10/L13; `TESTFLEX.Main` L59-L87
  `CORFLEXFLAT`; `COROUTINE_CALL` L79 `:CS "CORFLEX" :don't see this being called in code`).
  Requests `01`-`07` are therefore documented from the code, not from a confirmed caller list.
  The ACL-side caller needs SME confirmation.
- **`SD_PR_ERRFLG` meaning:** the literal values assigned are confirmed (see the specification),
  but their business meaning ("policy not found" vs "term not found") is inferred. Confirm with
  an SME.
- **`GI_BLD_*` inserted blocks:** `GI_BLD_HST_INQ` and `GI_BLD_PNDCR_INQ` were traced (each stores
  one `FBSCN` row if none exists). The other 8 inserted blocks used by `Frame70` were not read.
- **`GP_CORACL`, `GP_HISTORY_TYPE`, `GP_AMT_DISP`, `GP_CANCEL_REASONS`:** only their headers and
  first lines were read; their full code tables are not documented here.
- **`Frame90`:** orphan/debug frame; caller not found. Commented-out call in `Main` L13
  (`EX 90 AS ... (CORFLEXKAT)`).
- **`GC_01` into `SD_PR_ERRFLG`:** `Frame30` L13 moves the 2-character constant `GC_01` into a
  1-character field; the stored value (presumably `"1"`) was not verified at runtime.
- **FBSCN retention:** `Frame70` purges its `FBSCN` rows at the start of each call but not at the
  end, so rows remain until the next call for the same user/station/ispec. Retention outside this
  routine was not checked.
