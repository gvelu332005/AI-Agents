# AGENTS.md

## What this folder is
Documentation-only requirements for the FLEX **CORFLEX** coroutine (report) in
`Matrix_FLEXINT`. There is no application code here; the deliverable is documentation,
starting in
[ReadMe.md](ReadMe.md), with field-level detail in
[docs/CORFLEX-Specification.md](docs/CORFLEX-Specification.md) and
[docs/CORFLEX-Data-Sources.md](docs/CORFLEX-Data-Sources.md).

## Scope boundaries
- Only use FLEX code and database structures that pertain to `CORFLEX` and its `Frame*.Main`
  children, plus the shared `GP_*` / `GI_BLD_*` routines they call.
- Do not invent unconfirmed behavior; record it under
  [ReadMe.md#open-questions](ReadMe.md#open-questions).

## Researching the source
- Server `CR25ABSSQLBLD1`, database `Matrix_FLEXINT`, Windows authentication, **read-only SELECT**.
- Owners are in `dbo.OwnerNames`; source text is in `dbo.LinesOfCode` (`[LineNo]`, `[Text]`,
  `Owner` = owner `ABSId`); parsed commands are in `dbo.Commands`; data items in `dbo.DataItems`.
  Bracket-quote `[LineNo]`.
- Only the 11 `.Main` owners (`CORFLEX.Main`, `Frame10/11/20/30/40/50/60/61/70/90.Main`) have code.
  `ContainerReference` has no rows for CORFLEX, so file usage comes from `dbo.Commands`
  (`LDLGroup` = `DBAccess`, `LDLCommand` = `Determine`/`LookUp`) and the source text.
- Inserted blocks (`INS GI_BLD_*`) are not expanded in the `LinesOfCode` of the calling owner:
  the line numbers jump by the size of the inserted block.
- Text after a `:` in a source line is a comment, not a value.

## Conventions
- Keep the README structured as Overview / Scope / Requirements / Confirmed Frame Logic /
  Open Questions; put detail in `docs/`.
- Mark inferred meaning as inferred; keep confirmed code facts and interpretation separate.
