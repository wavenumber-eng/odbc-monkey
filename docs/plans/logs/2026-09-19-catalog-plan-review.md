+++
type = "plan_log"
id = "recursive-multi-format-catalog-review-2026-09-19"
plan_id = "recursive-multi-format-catalog"
step_id = "behavior-contract"
created = "2026-09-19T12:00:00-04:00"
+++

# Independent plan review

Two independent reviews were completed before committing this plan: one focused on governance and deliverability, and one on native implementation and test risk. The reviewers did not edit the repository.

Material findings incorporated into the plan:

- Standardized the driver and DbLib helper on canonical `cad-reference`; arbitrary helper-only primary-key overrides are deprecated because they cannot participate in catalog validation.
- Narrowed the JSON compatibility promise and required a pre-contract census plus fixtures for the intentional schema-union and key-validation differences.
- Reordered characterization before contract approval and the golden corpus before reader implementation.
- Added strict initial-load versus refresh diagnostics, ODBC table/column collision rules, key literal transport rules, worksheet eligibility, and portable cross-format scalar coverage.
- Required bounded native parsers, unsupported-feature policy, malformed-input and archive-expansion tests, and governed workbook fixtures.
- Required shared immutable snapshot ownership plus recovery tests for watcher overflow, restart, root recreation, and changes during a rebuild.
- Chose one reusable native catalog library and inspection CLI as the parity mechanism for the driver and Python DbLib helper; a manually maintained manifest is not a runtime integration boundary.
- Removed the implication that recursive JSON discovery had an independent release gate. All release readiness remains under this plan's signoff and closeout.
- Kept proposed design documents and ADRs provisional until implementation evidence and signoff justify accepted/current status.

This review refines a pending implementation plan. It does not satisfy or close the plan's final `external-review` step, which remains a post-implementation closeout check.
