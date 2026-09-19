+++
type = "plan"
id = "dev-std-adoption"
status = "active"
created = "2026-09-19"

[[steps]]
id = "bootstrap-dev-std"
title = "Pin wn-dev-std and establish plan governance"
status = "done"

[[steps]]
id = "baseline-audit"
title = "Record the full cpp-library compliance baseline"
status = "pending"
depends_on = ["bootstrap-dev-std"]

[[steps]]
id = "repository-foundation"
title = "Add the required repository governance and local build foundation"
status = "pending"
depends_on = ["baseline-audit"]

[[steps]]
id = "documentation-framework"
title = "Establish the standard documentation hierarchy and ownership"
status = "pending"
depends_on = ["repository-foundation"]

[[steps]]
id = "architecture-and-design"
title = "Document the existing architecture, setup, build, and design"
status = "pending"
depends_on = ["documentation-framework"]

[[steps]]
id = "contracts-and-traceability"
title = "Document contracts, requirements, decisions, artifacts, and vendors"
status = "pending"
depends_on = ["architecture-and-design"]

[[steps]]
id = "local-test-governance"
title = "Add local Rack, CTest, quality, and signoff governance"
status = "pending"
depends_on = ["contracts-and-traceability"]

[[steps]]
id = "full-profile-audit"
title = "Remove the partial-adoption ratchet and pass the full cpp-library audit"
status = "pending"
depends_on = ["local-test-governance"]

[[steps]]
id = "design-doc-intent-audit"
title = "Audit design docs, ADRs, and requirements against implementation"
status = "pending"
depends_on = ["full-profile-audit"]

[[steps]]
id = "test-runtime-impact-audit"
title = "Audit new test runtime impact"
status = "pending"
depends_on = ["full-profile-audit"]

[[steps]]
id = "external-review"
title = "Obtain independent external review"
status = "pending"
depends_on = ["full-profile-audit", "design-doc-intent-audit", "test-runtime-impact-audit"]

[[exit_criteria]]
id = "tooling"
title = "wn-dev-std is pinned and its configuration version matches the installed package"
status = "met"

[[exit_criteria]]
id = "full-conformance"
title = "The unscoped local dev-std audit passes for the cpp-library profile"
status = "pending"

[[exit_criteria]]
id = "documentation"
title = "Every required durable document describes the current repository rather than proposed driver changes"
status = "pending"

[[exit_criteria]]
id = "local-signoff"
title = "The documented local build, test, Rack, and quality signoff commands pass"
status = "pending"

[[exit_criteria]]
id = "driver-stability"
title = "No driver behavior, data-source behavior, or public interface changes are included"
status = "pending"

[[exit_criteria]]
id = "ci-deferred"
title = "No CI workflow is added or changed by this plan"
status = "pending"

[[exit_criteria]]
id = "design-doc-intent-audit"
title = "Design docs, ADRs, and requirements match implementation"
status = "pending"

[[exit_criteria]]
id = "test-runtime-impact-audit"
title = "New tests are listed and runtime impact is reviewed"
status = "pending"

[[exit_criteria]]
id = "external-review"
title = "Independent external review is complete"
status = "pending"
+++

# Bring odbc-monkey under wn-dev-std

## Objective

Adopt `wn-dev-std` as the repository's local development-governance contract, then bring the existing project into conformance one standard area at a time. This plan governs the adoption work itself; it does not redesign the ODBC driver.

The repository is governed as a `cpp-library` because its primary deliverable is the native Windows ODBC driver. The Python package is supporting tooling and remains inside the same repository boundary.

## Delivery sequence

### 1. Bootstrap plan governance

- Pin `wn-dev-std==2026.9.8` for Python 3.12+ in the development dependency group and generate a committed `uv.lock`. Keep the helper package's existing Python 3.11 compatibility; development-governance commands run under Python 3.12 or newer.
- Add `dev-std.toml` with the matching standard version, the `cpp-library` profile, and `docs/plans` as the initial enabled scope.
- Validate this plan with `uv run dev-std audit .`.
- Keep partial adoption explicit: passing the plan scope is not yet full project conformance.

### 2. Capture the compliance baseline

- Run `uv run dev-std audit . --scope all --format json` and preserve the actionable findings in a plan log.
- Group findings by repository foundation, documentation, build, tests, artifacts, vendors, and release governance.
- Confirm that `cpp-library` remains the correct profile and record an ADR if mixed native/Python layout needs a lasting exception or boundary decision.
- Convert the audit into small, reviewable work batches; do not satisfy checks with empty placeholder documents.

The initial reconnaissance already identifies missing root governance files, documentation roots, Rack configuration, build documentation, ADRs, requirements, domain registration, and artifact governance. The tracked prebuilt DLL and vendored `nlohmann/json.hpp` must be explicitly cataloged rather than silently grandfathered.

### 3. Establish the repository foundation

- Add and tailor the required root governance files, including `AGENTS.md`, `CONTRIBUTING.md`, `CHANGELOG.md`, and `.gitattributes`.
- Establish the standard local C++ tooling contract (`.clang-format`, `.clang-tidy`, CMake presets using Ninja, compile commands, warning policy, and CTest registration) while preserving existing supported build paths until their replacements are verified.
- Add `signoff.toml` and the Rack suite structure needed for local governance.
- Define the public/private native header boundary and document how the Python helper participates in build and test workflows.

### 4. Work through the standard documentation

Create and substantively populate the required documentation in this order:

1. `docs/setup.html`, `docs/build/index.html`, and `docs/test-strategy.html` so contributors can reproduce the local environment, build, and tests.
2. `docs/architecture.html` and `docs/design/cpp-standard.html` to describe the current native driver, Python helper, process boundaries, data flow, ownership, and C++ conventions.
3. Focused design documents for existing behavior only: ODBC surface, JSON source discovery/parsing, normalized catalog construction, configuration, and installer/registration behavior.
4. `docs/contracts/` for machine-readable or precisely specified public surfaces and compatibility expectations.
5. `docs/requirements/`, `docs/adr/`, and the domain registry to connect intent, ownership, decisions, implementation, and verification.
6. `docs/governance/artifacts.toml`, vendor inventory, and release documentation for the prebuilt DLL, generated outputs, `nlohmann/json`, packaging, and versioning.

Each document must distinguish observed behavior from future ideas. Unknowns found while documenting are recorded as follow-up work; they are not resolved by changing the driver under this plan.

### 5. Add local verification and reach full conformance

- Register the existing native and Python tests under the documented local test strategy.
- Add only the governance and characterization tests necessary to prove current behavior and documentation claims.
- Establish local formatting, static-analysis, complexity, audit, and signoff commands with realistic runtime budgets.
- Expand `enabled_scopes` only when the newly enabled scope passes. Remove `enabled_scopes` after the complete `cpp-library` profile passes.
- Perform the required design-document intent audit, test-runtime impact audit, and independent review before closeout.
- Distill durable outcomes into governed documents, then remove this temporary plan and its logs according to the standard closeout process.

## Explicitly out of scope

- GitHub Actions or any other CI work, including adopting or revisiting pull request #2.
- Recursive file discovery.
- CSV, XLSX, or XLS ingestion and selection of spreadsheet libraries.
- Provider abstraction, mixed-provider scanning, catalog normalization changes, or source precedence rules.
- Changes to ODBC behavior, schemas, installation behavior, public interfaces, or compatibility.
- Release automation beyond documenting the current process and the future requirements.

Those driver and data-source improvements may be captured in a dependent pending plan, but they will not be implemented until this standards-adoption plan is complete.

## Working rules

- No audit item is closed with an empty placeholder or an unverified claim.
- Existing behavior is characterized before it is documented as contractual.
- Each scope expansion must pass locally before it becomes the repository default.
- Any exception to the standard requires a written rationale, owner, and retirement condition.
- CI remains a separate future decision even if the standard recommends a CI gate.
