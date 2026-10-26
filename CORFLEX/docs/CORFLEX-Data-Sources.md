# CORFLEX — Data Sources

Source: ABSuite Matrix DB (`Matrix_FLEXINT` on `CR25ABSSQLBLD1`), `dbo.Commands` (`DBAccess` rows),
the LDL+ source of `CORFLEX.Main` and `Frame10/11/20/30/40/50/60/61/70/90.Main`, and
`dbo.DataItems` (type/length). See [CORFLEX-Specification.md](CORFLEX-Specification.md) for how
each field is used in the control flow.

Entity descriptions are the container descriptions on file in Matrix. Types: `AsString`,
`AsNumber`, `AsSignedNumber`; length is in characters/digits and the decimals are shown after the
length where there are any.

Field enumeration is complete for the entities read directly by the 11 CORFLEX programs. The
entities read only through `GI_BLD_*` inserted blocks (`FDAYC`, `CMNTS`, `EFACT`, `EFHST`, `PCASH`,
`PNDCR`) are listed with profiles but not field by field (see [Coverage note](#coverage-note)).

## Entity summary

| Entity | Description on file | Profiles used | Read by |
|---|---|---|---|
| `FBPOL` | Flex Policy Master | `FBPOLP00`, `FBPOLP03` | `Frame10`, `Frame30`, `Frame60`, `Frame70` |
| `FSTAT` | Statistical Records | `FSTATP00` | `Frame20`, `Frame30` |
| `FCNCL` | Final Cancel-to-Codes | `FCNCLP00` | `Frame10`, `Frame70` |
| `EFMST` | EFT Master | `EFMSTP01` | `Frame11`, `Frame60`, `Frame70` |
| `ACTNM` | Account Name/Address | direct `LookUp` by account | `Frame10` |
| `ECASH` | Flex Entry Cash Structure | direct `LookUp` by control number; `ECASHP02`-`P05` | `Frame40`, `Frame70` |
| `FHRAS` | Hurricane Assmt Structure | direct `LookUp` by number | `Frame50` |
| `FHIST` | Flex Activity History Structure | `FHISTP00` | `Frame60`, `Frame70` |
| `PDHST` | Payment History | `PDHSTP02` | `Frame61` |
| `NOTFY` | Notify NmAddr (Mort-n-Other) | `NOTFYP00` | `Frame70` |
| `FBSCN` | Screen Info | `FBSCNP01`, `FBSCNP02` | `Frame70` (and `GI_BLD_*`); **written** |
| `FDAYC` | Bill Day Change/WO Rqst Bill | `FDAYCP00`, `FDAYCP02` | `Frame70` |
| `CMNTS` | Comment Structure | `CMNTSP00`, `CMNTSP02` | `Frame70` |
| `EFACT` | EFT Activity | `EFACTP01` | `Frame70` |
| `EFHST` | EFT History | `EFHSTP01` | `Frame70` |
| `PCASH` | Pending Cash - Bill Matrix | `PCASHP02`, `PCASHP03` | `Frame70` |
| `PNDCR` | Pending Credits | `PNDCRP01`, `PNDCRP02` | `Frame70` |

## `FBPOL` — Flex Policy Master

Queried via `FBPOLP00 (COB, PREFIX, POLICY, EXP_DATE)` and `FBPOLP03 (POLICY, then three further key
parts set to high values)` (policy-number alternate key, `Frame60` only). Fields below are those the
CORFLEX programs read.

| Field | Type | Length | Used by |
|---|---|---|---|
| `COB` | AsNumber | 4 | key; `Frame10` branch remap (positions 3-4); `Frame60` max installments |
| `PREFIX` | AsString | 3 | key |
| `POLICY` | AsString | 8 | key |
| `EXP_DATE` | AsNumber | 8 | key; `Frame30` loss-date term test; `Frame60` EFT validity |
| `EFF_DATE` | AsNumber | 8 | `Frame10` -> `SD_PR_EFF_DT`; `Frame30` loss-date term test |
| `ACCOUNT` | AsNumber | 10 | `Frame10`, `Frame11`, `Frame60`, `Frame70` (also `ACTNM`/`EFMST` key) |
| `AGENT_NBR` | AsNumber | 6 | `Frame10` -> `SD_PR_AGENT`; `Frame61` (`PDHST` key) |
| `BILLG_DAY` | AsNumber | 2 | `Frame10`; `Frame70` L19 |
| `POL_STATUS` | AsString | 2 | `Frame10` status table; `Frame70` status text |
| `TOTAL_DUE` | AsSignedNumber | 10,2 | `Frame10`; `Frame70` L14, L26 |
| `BILLAMTDUE` | AsSignedNumber | 10,2 | `Frame10` -> minimum due; `Frame70` L22 |
| `BILLDUE_DT` | AsNumber | 8 | `Frame10`; `Frame70` (`99999999` = none) |
| `PCN_SND_DT` | AsNumber | 8 | `Frame10` (status `IC`) |
| `FNLCANC_DT` | AsNumber | 8 | `Frame10`, `Frame70` (statuses `PC`, `CC`) |
| `LAPSE_DATE` | AsNumber | 8 | `Frame10`; `Frame70` L31, 182-day rule |
| `NSF_POLICY` | AsString | 1 | `Frame10` NSF flag/prefix |
| `PCN_NBR` | AsNumber | 2 | `Frame10`, `Frame60`, `Frame70` L9 |
| `PREM_STATE` | AsNumber | 2 | `Frame10`, `Frame60`, `Frame70` L23 |
| `PYMTS_RCVD` | AsNumber | 2 | `Frame10`, `Frame60`, `Frame70` L5 |
| `INS_NAME` | AsString | 27 | `Frame10`, `Frame60` |
| `CITY` / `STATE_ABBR` | AsString | 30 / 2 | `Frame10` -> mailing city/state |
| `CAT_ID` | AsString | 4 | `Frame10` (overwritten with `00`) |
| `STAT_RELSD` | AsString | 1 | `Frame10` -> `SR_PR_STATSH` |
| `TERM_PAID` | AsSignedNumber | 10,2 | `Frame10`; `Frame70` L30 |
| `PNDG_CRDT` | AsSignedNumber | 10,2 | `Frame10`; `Frame70` L5 |
| `WHOPAY_IND` | AsString | 1 | `Frame11`, `Frame60`, `Frame70` (`I`, `M`, `O`) |
| `BOND_FLG` | AsString | 1 | `Frame60`, `Frame70` (`Y` = bond) |
| `NSF_NBR` | AsNumber | 2 | `Frame60`, `Frame70` L8 |
| `PYMTS_LATE` | AsNumber | 2 | `Frame60`, `Frame70` L7 |
| `TERMINMOS` | AsNumber | 4 | `Frame60`, `Frame70` (term in months) |
| `CANC_REASN` | AsNumber | 2 | `Frame60`, `Frame70` L23 (`GP_CANCEL_REASONS`) |
| `PTD_POLICY` | AsNumber | 8 | `Frame70` paid-to date, cancelled status text |
| `BILLS_SENT` | AsNumber | 2 | `Frame70` L3, L4 |
| `ACTG_CHRG` | AsSignedNumber | 10,2 | `Frame70` L4 |
| `NBR_MISDUE` | AsNumber | 2 | `Frame70` L6 |
| `PRV_COB` / `PRV_PREFIX` / `PRV_POLNO` / `PRV_EXP_DT` | AsNumber 4 / AsString 3 / AsString 8 / AsNumber 8 | | `Frame70` L6, policy chain walk |
| `RNR_COB` / `RNR_PREFIX` / `RNR_POLNO` | AsNumber 4 / AsString 3 / AsString 8 | | `Frame70` L8 |
| `LSTBL_DATE` | AsNumber | 8 | `Frame70` L9 |
| `LSTBL_TOTL` | AsSignedNumber | 10,2 | `Frame70` L10 |
| `LSTBL_ENTY` | AsString | 1 | `Frame70` L32 |
| `PRV_BLDEND` | AsSignedNumber | 10,2 | `Frame70` L11 |
| `REINS_PREM` | AsSignedNumber | 10,2 | `Frame70` L13; cancelled text |
| `CANC_PREM` | AsSignedNumber | 10,2 | `Frame70` L20; cancelled text |
| `ROLLOVER` | AsString | 1 | `Frame70` L15 |
| `SRVCHG_DUE` / `SRVCHG_PD` | AsNumber | 4,2 | `Frame70` L16, L17 |
| `FEE_DUE_BR` / `FEE_PD_BR` | AsSignedNumber | 6,2 | `Frame70` L18, L19, L26 |
| `FEE_DUE_LT` / `FEE_PD_LT` | AsSignedNumber | 6,2 | `Frame70` L20, L21, L26 |
| `COMMUPFRNT` | AsString | 1 | `Frame70` L24 |
| `PERS_COMM` | AsString | 1 | `Frame70` L24 |
| `SYSORIGIN` | AsString | 3 | `Frame70` L24 |
| `WO_PREM` | AsSignedNumber | 10,2 | `Frame70` L27; cancelled text |
| `TOTAL_ENDS` | AsSignedNumber | 10,2 | `Frame70` L27 |
| `NWRNR_PREM` | AsSignedNumber | 10,2 | `Frame70` L28 |
| `TERM_TOTAL` | AsSignedNumber | 10,2 | `Frame70` L29 |
| `WCCODED` | AsString | 1 | `Frame70` L31 |
| `WRITEOFF` | AsString | 1 | `Frame70` L32 (`01` = `Y`) |
| `NSF_PYMTS` | AsSignedNumber | 10,2 | `Frame70` L18 |

## `FSTAT` — Statistical Records

Queried via `FSTATP00 (COB, PREFIX, POLICY, EXP_DATE)` in `Frame20` and `Frame30`.

| Field | Type | Length | Used by |
|---|---|---|---|
| `STAT_IMAGE` | AsString | 720 | Moved whole into `GG_STAT_RECORD`; the record layout (`GN_STAT_REC_TYP`, `GN_STAT_TRANCODE`, `GA_STAT_MLCV`, `GA_STAT_DEC_CODE`, `GN_STAT_SUBLINE`, `GN_STAT_CLASS2`, `GA_STAT_LOC_UNIT`, `GA_STAT_BLDG`, `GN_STAT_AMT_INS`, `GN_STAT_LOC_ZIP`, `GN_STAT_GROSS`, `GN_STAT_REIN_CO`, `GN_STAT_REIN_PCT`) is applied by the program |

`GG_STAT_RECORD` is a shared global group; its field positions were not enumerated here.

## `FCNCL` — Final Cancel-to-Codes

Queried via `FCNCLP00 (COB, PREFIX, POLICY, EXP_DATE)` in `Frame10` and `Frame70` for status `CC`.

| Field | Type | Length | Used by |
|---|---|---|---|
| `CANC_EFFDT` | AsNumber | 8 | `SD_ACL_DATE` / status text (`MM-DD-CCYY`) |

## `EFMST` — EFT Master

Queried via `EFMSTP01 (record type, key)`: type `01` + account (insured-billed), type `02` +
COB/prefix/policy.

| Field | Type | Length | Used by |
|---|---|---|---|
| `BGN_DATE` | AsNumber | 8 | begin date; `Frame11`/`Frame70` compare to today, `Frame60` to policy expiration |
| `END_DATE` | AsNumber | 8 | end date; same |
| `HOLD_FLG` | AsString | 1 | `Y` -> held |
| `HOLD_DATE` | AsNumber | 8 | `Frame60` hold test; `Frame70` hold date |
| `HOLD_REASN` | AsString | 20 | `Frame70` -> `GA_EFT_HOLDREASN` |
| `EFT_PAC_ID` | AsString | 1 | `R` = RBP |
| `DPSTR_TYP` | AsString | 1 | `Frame60` compared to `WHOPAY_IND` |
| `SETUP_DATE` | AsNumber | 8 | `Frame70` L28 |
| `LSTCHG_DT` | AsNumber | 8 | `Frame70` (`GN_EFT_LSTCHGDT`) |
| `BANK_ACCT` | AsString | 17 | `Frame70` positions 8-11 -> `EFT_INFO` (see [sensitive fields](#sensitive-fields)) |

## `ACTNM` — Account Name/Address

`LU FBPOL.ACCOUNT ACTNM` in `Frame10` (success when the status is spaces).

| Field | Type | Length | Used by |
|---|---|---|---|
| `ALERT_DYS` | AsNumber | 2 | -> `SD_PR_ALRTDYS` |
| `ELCTR_BILL` | AsString | 1 | -> `SD_PR_EBILL` |
| `EMAIL_ADDR` | AsString | 50 | -> `SD_PR_EMLADDR` |

## `ECASH` — Flex Entry Cash Structure

`LU SG_PR_COBRPO ECASH` (control-number request) in `Frame40`; profiles `ECASHP02`-`P05`
(by account, by policy, by account for transfers, by policy for transfers) in `Frame70`.

| Field | Type | Length | Used by |
|---|---|---|---|
| `AGENT_NBR` | AsNumber | 6 | `Frame40` -> `SD_PR_AGENT` |
| `STATE_NBR` | AsNumber | 2 | `Frame40` -> `SD_PR_PRMST` |
| `INS_NAME` | AsString | 27 | `Frame40` -> `SD_PR_INSNM` |
| `CASH_SUBTY` | (not enumerated) | | `Frame70` filter `<> 04` for the transfer insert |

Other `ECASH` fields are used only inside the `GI_BLD_ECASH_*` / `GI_BLD_PNDG_ECASH` blocks.

## `FHRAS` — Hurricane Assmt Structure

`LU SD_HA_HANBR FHRAS` in `Frame50`.

| Field | Type | Length | Description on file | Used by |
|---|---|---|---|---|
| `HA_NAME1` | AsString | 21 | Hurr Asmt Long Name | -> `SD_HA_DESC` |

## `FHIST` — Flex Activity History Structure

Queried via `FHISTP00 (COB, PREFIX, POLICY[, ACT_DATE, SEQ_NBR])`.

| Field | Type | Length | Used by |
|---|---|---|---|
| `COB` / `PREFIX` / `POLICY` | AsNumber 4 / AsString 3 / AsString 8 | | key |
| `EXP_DATE` | AsNumber | 8 | term filter (`Frame60`, `Frame70`) |
| `ACT_DATE` | AsNumber | 8 | history date; `PDHST` key |
| `SEQ_NBR` | AsNumber | 3 | key |
| `ACT_TYP` | AsNumber | 2 | activity type: drives type/count/balance rules |
| `ACT_SUBTYP` | AsNumber | 3 | payment method / transfer subtype |
| `ACT_DESC` | AsString | 55 | transfer-from capture (`Frame61`); `PDHST` control (first 15) |
| `ACT_AMT1` | AsSignedNumber | 10,2 | history amount |
| `ACT_AMT2` | AsSignedNumber | 10,2 | min/source amount |
| `ACT_AMT4` / `ACT_AMT5` | AsSignedNumber | 6,2 | balance computation and `*` marker |
| `TRN_EFFDUE` | AsNumber | 8 | effective-due date (`99999999` / zeros = none) |
| `AUD_PAYPLN` | AsNumber | 1 | `GG_FHIST_INFO.GN_ACT_AUDBL` |
| `BILLENTITY` | AsString | 1 | `GA_BILLED_ENTY` |
| `POL_V_ACCT` | AsString | 1 | `GA_POL_VS_ACCT` |
| `ACCOUNT` | AsNumber | 10 | `FBSCN.ACCOUNT` (via insert) |

## `PDHST` — Payment History

`PDHSTP02 (FBPOL.AGENT_NBR, FHIST.ACT_DATE, first 15 characters of FHIST.ACT_DESC)` in `Frame61` for
activity type 46.

| Field | Type | Length | Used by |
|---|---|---|---|
| `USERCODE` | AsString | 17 | non-blank -> `SD_PAYTYP` = `ACH` ("other than monthly eft") |
| `CNTRL_NBR` | AsString | 15 | matched by the 3rd key |

## `NOTFY` — Notify NmAddr (Mort-n-Other)

`NOTFYP00 (COB, PREFIX, POLICY, EXP_DATE, type) BACK` in `Frame70`; type `M` (mortgagee), `B` (bond obligee)
or `O` (other payor).

| Field | Type | Length | Used by |
|---|---|---|---|
| `NAME` / `NAME2` | AsString | 50 / 50 | name lines |
| `ADDRLINE1` / `ADDRLINE2` | AsString | 35 / 35 | address lines |
| `CITY` | AsString | 30 | `CITY STATE ZIP` line |
| `STATE_ABBR` | AsString | 2 | same |
| `ZIPCODE` | AsString | 10 | same |
| `LOAN_ID` | AsString | 15 | `Loan Number: ` line (non-bond only) |

`SG_MORT.NAMEADDRn` / `SG_PAY.NAMEADDRn` are 36 characters, so the 50-character names are truncated
to 36 in the response.

## `FBSCN` — Screen Info scratch file

Description on file: "Screen Info". The only entity CORFLEX writes. `Frame70` and the inserted
`GI_BLD_*` blocks use it as a per-session work table; the key starts with `USERCODE`, `INITSTN`
(station) and `ISPECNM` (`ACLBL`).

| Field | Type | Length | Meaning in CORFLEX |
|---|---|---|---|
| `USERCODE` | AsString | 17 | `GLB.USERCODE` |
| `INITSTN` | AsString | 17 | `GLB.Station` |
| `ISPECNM` | AsString | 5 | `ACLBL` |
| `FILENM` | AsString | 5 | source file (`FBPOL`, `FHIST`, `PNDCR`, ...) |
| `I_POL_ACCT` | AsString | 1 | `P` (policy) |
| `I_KEYFLD` | AsString | 15 | COB + prefix + policy key |
| `ACCOUNT` | AsNumber | 10 | account |
| `ACT_DATE` | AsNumber | 8 | activity date (or `FBPOL.EXP_DATE`) |
| `COB` / `PREFIX` / `POLICY` / `COB_POLICY` | AsNumber 4 / AsString 3 / AsString 8 / AsString 15 | | policy identity |
| `REC_TYP` / `REC_SUBTYP` / `SEQ_NBR` | AsNumber 2 / 2 / 3 | | source activity type/subtype/sequence |
| `CNTRL_NBR` | AsString | 15 | control number (cash/credits) |
| `AMOUNT` | AsSignedNumber | 10,2 | amount; for `FHIST` rows updated with the running balance |
| `ENTRY_USER` | AsString | 17 | spaces or `GC_FLEX` |
| `UNIQTRAN` | AsNumber | 12 | `GLB.UNIQUE` per request |
| `TRANTIME` | AsNumber | 8 | not set by the traced code |

Write operations (all in `Frame70` / inserted blocks): `Purge` at the start of each call (L2-L7, L1036-L1040),
`Store` per source row, and `FL` of `AMOUNT` (L2847). No end-of-call purge was found, so rows persist
until the next call for the same user/station/ispec.

## Entities read only via inserted blocks

`FDAYC`, `CMNTS`, `EFACT`, `EFHST`, `PCASH`, `PNDCR` (and the cash-side fields of `ECASH`) are only
read in `Frame70`'s keyed loops and inside the `GI_BLD_*` blocks, which copy the record into an
`FBSCN` row. Profiles and keys are in the [specification](CORFLEX-Specification.md#phase-5--policy-chain-and-scratch-rows-l1031-l2455).

## Sensitive fields

CORFLEX returns, and in one place stages, personal and financial data. Listed so a rewrite can
decide on masking, logging and retention.

| Data | Where it comes from | Where it is returned or stored |
|---|---|---|
| Insured name, mailing city/state | `FBPOL.INS_NAME`, `CITY`, `STATE_ABBR` | `SD_PR_INSNM`, `SD_PR_MLGCITY`, `SD_PR_MLGST`; `SD_PM_INSNM` |
| Email address, alert days, e-bill flag | `ACTNM` | `SD_PR_EMLADDR`, `SD_PR_ALRTDYS`, `SD_PR_EBILL` |
| EFT/bank information | `EFMST.BANK_ACCT` (positions 8-11) | `SG_ACLBL_PARAM.EFT_INFO` (`AUTO-nnnn`, `AUTO-nnnnHELD`) |
| EFT status, hold, begin/end/setup dates | `EFMST` | `SD_PR_EFT_FLG`, `SD_PR_EFT_RBP`, "Important Info" L28 |
| Mortgagee / bond obligee / other payor names and addresses, loan number | `NOTFY` | `SG_MORT`, `SG_PAY` |
| Account numbers, previous/renewal policy numbers | `FBPOL` | `SG_IMPINFO` L6-L8, L13-L15, L33; `ACCTNBR` |
| Amounts due, payments, fees, balances | `FBPOL`, `FHIST` | `SG_PR_RQST`, `SG_IMPINFO`, `SG_INFO`, `SG_POLICIES` |
| Reinsurance company and percent | `FSTAT` | `SD_PR_REINCO`, `SD_PR_REINSPC` |
| Staged copies of the above identifiers and amounts | `FHIST`, `PNDCR`, ... | `FBSCN` rows (persist until the next call for the same user/station) |

No SSN, date of birth or full bank account number is read or returned by the traced code. The
`BANK_ACCT` positions used (8-11) are four characters of a 17-character field; whether that is
the last four digits depends on how the field is padded and was not verified.

## Coverage note

Not enumerated field by field:

- `GG_STAT_RECORD` (the `FSTAT.STAT_IMAGE` layout) and `GG_FHIST_INFO` / `GP_HISTORY_TYPE` code tables.
- `FDAYC`, `CMNTS`, `EFACT`, `EFHST`, `PCASH`, `PNDCR`, and the cash fields of `ECASH`.
- `GP_CORACL` / `GWG_ACL_PFILE` (the ACL policy data used for `Billing Center Account`).
- 8 of the 10 `GI_BLD_*` blocks (`GI_BLD_FDAYC_CMNT`, `GI_BLD_CMNT_INQ`, `GI_BLD_ECASH_CMNT`,
  `GI_BLD_PNDG_EFACT`, `GI_BLD_PNDG_EFHST`, `GI_BLD_PNDG_ECASH`, `GI_BLD_PNDG_PCASH`,
  `GI_BLD_ECASH_TRFR`).
