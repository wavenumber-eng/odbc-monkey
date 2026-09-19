+++
type = "plan"
id = "recursive-multi-format-catalog"
status = "pending"
created = "2026-09-19"
depends_on = ["dev-std-adoption"]

[[steps]]
id = "characterization-tests"
title = "Characterize current JSON, catalog, ODBC metadata, query, and refresh behavior"
status = "pending"

[[steps]]
id = "behavior-contract"
title = "Approve part identity, source discovery, tabular mapping, diagnostics, and compatibility contracts"
status = "pending"
depends_on = ["characterization-tests"]

[[steps]]
id = "catalog-core"
title = "Adopt the normalized catalog and feed it through the existing JSON reader"
status = "pending"
depends_on = ["behavior-contract"]

[[steps]]
id = "recursive-json-discovery"
title = "Implement recursive JSON discovery and recursive refresh for issue 4"
status = "pending"
depends_on = ["catalog-core"]

[[steps]]
id = "tabular-source-contract"
title = "Finalize the shared CSV and spreadsheet row-to-part contract"
status = "pending"
depends_on = ["recursive-json-discovery"]

[[steps]]
id = "golden-corpus"
title = "Create the canonical cross-format dataset and expected catalog manifest"
status = "pending"
depends_on = ["tabular-source-contract"]

[[steps]]
id = "csv-reader"
title = "Add recursive CSV ingestion"
status = "pending"
depends_on = ["golden-corpus"]

[[steps]]
id = "xlsx-reader"
title = "Select, vendor, and integrate a read-only XLSX reader"
status = "pending"
depends_on = ["golden-corpus"]

[[steps]]
id = "xls-reader"
title = "Vendor and integrate libxls for legacy XLS files"
status = "pending"
depends_on = ["golden-corpus"]

[[steps]]
id = "mixed-source-refresh"
title = "Make mixed-format catalog construction and live refresh deterministic and atomic"
status = "pending"
depends_on = ["csv-reader", "xlsx-reader", "xls-reader"]

[[steps]]
id = "helper-docs-and-release"
title = "Update DbLib generation, contracts, documentation, examples, and release notes"
status = "pending"
depends_on = ["mixed-source-refresh"]

[[steps]]
id = "focused-signoff"
title = "Run compatibility, integration, installation, and local governance signoff"
status = "pending"
depends_on = ["helper-docs-and-release"]

[[steps]]
id = "design-doc-intent-audit"
title = "Audit design docs, ADRs, and requirements against implementation"
status = "pending"
depends_on = ["focused-signoff"]

[[steps]]
id = "test-runtime-impact-audit"
title = "Audit new test runtime impact"
status = "pending"
depends_on = ["focused-signoff"]

[[steps]]
id = "external-review"
title = "Obtain independent external review"
status = "pending"
depends_on = ["focused-signoff", "design-doc-intent-audit", "test-runtime-impact-audit"]

[[exit_criteria]]
id = "recursive-discovery"
title = "Root and nested supported files load and refresh with folder names having no classification meaning"
status = "pending"

[[exit_criteria]]
id = "automatic-readers"
title = "DataSource alone discovers JSON, CSV, XLSX, and XLS files without a required provider setting"
status = "pending"

[[exit_criteria]]
id = "json-compatibility"
title = "Existing JSON shape, schematic-version selection, values, classification, connection strings, and query semantics remain compatible except for documented validation and schema-union changes"
status = "pending"

[[exit_criteria]]
id = "tabular-parts"
title = "Every valid spreadsheet or CSV data row becomes one catalog part under the documented mapping contract"
status = "pending"

[[exit_criteria]]
id = "catalog-consistency"
title = "All readers produce one deterministic schema-unioned catalog and Parts view"
status = "pending"

[[exit_criteria]]
id = "cross-format-parity"
title = "Equivalent JSON, CSV, XLSX, and XLS fixtures produce the same normalized catalog snapshot"
status = "pending"

[[exit_criteria]]
id = "part-key-integrity"
title = "Every record has a stable Altium lookup key that is unique across the complete merged catalog"
status = "pending"

[[exit_criteria]]
id = "cache-and-locking"
title = "Source files have no persistent handles and a refresh atomically replaces the cached catalog"
status = "pending"

[[exit_criteria]]
id = "dependency-governance"
title = "Every vendored library is pinned, licensed, inventoried, reproducible, and covered by focused tests"
status = "pending"

[[exit_criteria]]
id = "resource-safety"
title = "Every native reader enforces documented resource limits and rejects unsafe or unsupported workbook features"
status = "pending"

[[exit_criteria]]
id = "helper-parity"
title = "DbLib generation and examples discover the same classification tables as the driver"
status = "pending"

[[exit_criteria]]
id = "signoff"
title = "The full local dev-std audit and focused native and Python signoff pass"
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

# Recursive, multi-format part catalog

## Objective

Make one directory path behave as a small, cached component catalog. The driver recursively discovers every supported source file under that root, selects a reader by extension, normalizes its records into one catalog, and exposes the catalog through the existing ODBC tables and `Parts` view.

This plan incorporates the request in [GitHub issue #4](https://github.com/wavenumber-eng/odbc-monkey/issues/4) and the follow-up design discussion covering JSON, CSV, XLSX, XLS, multiple parts per spreadsheet, and provider-free configuration. Implementation is blocked on `dev-std-adoption`; planning it now does not authorize starting driver changes early.

## Decisions already made

### Configuration and discovery

- `DataSource=<root-directory>` remains the only required source configuration.
- Discovery is recursive by default for `.json`, `.csv`, `.xlsx`, and `.xls`, matched case-insensitively.
- There is no required `Provider` setting. An internal extension-to-reader registry chooses the parser, and a single scan may use multiple readers.
- Directories are organizational only. A path such as `json/capacitor/vendor/part.json` does not create a table or classification.
- Classification comes from the record's `classification` field. Missing or blank classification continues to route to `orphanage`.
- Unsupported extensions, Excel temporary files such as `~$*.xlsx`, and non-regular filesystem entries are ignored. Symlink and junction behavior must be explicit and cycle-safe; the default is not to follow them.

### Record model

- The catalog owns normalized tables, columns, rows, source provenance, and diagnostics. ODBC query execution consumes that catalog and does not know which reader produced a row.
- JSON remains one logical part per file using the existing `versions` wrapper. The driver selects the latest UUIDv7 entry, which represents the latest schematic version for that part; it is not a version of the part itself.
- CSV, XLSX, and XLS are tabular sources: the first nonempty row supplies field names and each subsequent nonempty data row is one part. There is no title-row guessing or automatic header repair.
- For workbooks, consider visible worksheets independently and ignore hidden worksheets with a diagnostic. A worksheet qualifies only when its first nonempty row has nonblank, noncolliding headers and at least one header resolves through the part-key alias registry. Empty and nonqualifying worksheets follow the frozen diagnostics policy rather than being partially guessed.
- Folder and worksheet names do not classify parts. Every row uses its own `classification` cell, with `orphanage` as the fallback.
- Values cross the source-reader boundary as canonical text plus null/empty state. The tabular contract must specify numbers, booleans, dates, formulas and cached results, Unicode, blank cells, duplicate headers, and trimming before reader work begins.
- Table schemas are the deterministic union of fields across all records in that table. A field introduced by a later file or row must not be discarded.
- Records are not silently overwritten or deduplicated. Stable source-relative path, worksheet, and row provenance provides deterministic ordering and actionable duplicate diagnostics without adding public ODBC columns.

### Part identity and Altium lookup

- A source record has two different identities that must not be conflated:
  - the **part key** is stable across edits and links a placed Altium component back to one logical catalog record;
  - a JSON UUIDv7 `id` identifies a schematic version and is used only to select the latest schematic representation from the existing `versions` array.
- CSV and spreadsheet rows have no intrinsic stable identifier. File path, worksheet name, and row number are provenance, not part keys; generating a key from them would break links after a rename, move, sort, or inserted row.
- The current DbLib generator's `--primary-key` option defaults to `cad-reference` and writes a `UserWhereText` lookup using that field for every classification table and for `Parts`. Preserve `cad-reference` as the default part-key field for compatibility.
- In the default zero-configuration mode, resolve the part key through the documented alias policy below and project the result into a canonical `cad-reference` field. The generated DbLib always looks up this canonical field.
- Standardize the merged catalog and generated DbLib on canonical `cad-reference`; do not add a second key configuration channel. Deprecate arbitrary `--primary-key` values because the current option changes only DbLib `UserWhereText` and cannot inform driver-side projection or uniqueness validation. Continue accepting the default spelling, and fail a non-default override with a migration diagnostic rather than producing a mismatched catalog.
- Every JSON record and every CSV/XLSX/XLS data row must resolve to a nonblank part key. A reader may not synthesize one from provenance or unrelated data.
- The part key must be unique across the complete merged catalog, not merely within one file, worksheet, classification table, or source format. Validation must use the same case and text-comparison rules as ODBC equality queries.
- Uniqueness applies to logical catalog records. Exposing one record through both its classification table and the `Parts` projection does not create two logical parts. Characterization must verify how Altium behaves when both views are enabled in one DbLib; if overlapping enabled views make post-placement lookup ambiguous, the generator must stop enabling both by default.
- Two separate driver connections may reuse the same key values because they are separate catalogs. Once sources are combined under one `DataSource` root, their keys share one namespace.
- A repeated manufacturer part number is permitted when it represents distinct internal parts, but automatic MPN fallback will then raise a duplicate-key error. Such records need explicit distinct `cad-reference` or IPN values.
- Composite Altium Where-clause keys are technically possible but are not the default. Supporting them requires a separate design decision covering stable mapped parameters, DbLib generation, ODBC predicate correctness, and update behavior.

Part-key values must also satisfy the lookup transport contract:

- Support Unicode and document the effective ODBC/Altium length limit.
- Preserve meaningful internal whitespace after trimming only leading/trailing whitespace.
- Support apostrophes by implementing and testing standard doubled-quote SQL literal escaping/parsing through generated DbLib lookup, or reject such values with an explicit preflight diagnostic if Altium cannot round-trip them.
- Treat `%` and `_` literally for equality lookup; wildcard semantics apply only to an explicit `LIKE` query.
- Use the exact same case-folding and comparison routine for duplicate detection and ODBC equality matching.

#### Default key projection

Header matching is case-insensitive. Before matching, trim the header and remove spaces, hyphens, underscores, periods, and other non-alphanumeric separators. This makes `Manufacturer Part Number`, `manufacturer_part_number`, and `MANUFACTURER-PART-NUMBER` the same spelling without fuzzy matching.

Use the first nonblank value from the following ordered groups:

| Priority | Meaning | Accepted normalized header tokens |
|---:|---|---|
| 1 | Existing CAD reference | `cadreference`, `cadref` |
| 2 | Internal part number | `ipn`, `internalpartnumber`, `internalpn`, `companypartnumber`, `companypn` |
| 3 | Generic part number | `partnumber`, `pn` |
| 4 | Manufacturer part number | `mpn`, `manufacturerpartnumber`, `manufacturerpn`, `mfrpartnumber`, `mfrpn`, `manfpartnumber`, `manfpn` |

`manf_p_n` therefore normalizes to the documented `manfpn` token. The alias registry is a small reviewed data table shared by every reader, not format-specific code.

Projection rules are deliberately simple:

- Ignore blank candidates and trim leading/trailing whitespace from the selected key value.
- Preserve every original source field; projection only adds or fills canonical `cad-reference`.
- If two columns within the same semantic group contain different nonblank values, reject the record as ambiguous and report both headers and values.
- Values in lower-priority groups may legitimately differ and remain available as ordinary fields; for example, IPN and MPN normally differ.
- Record the selected source header in internal provenance so validation output can explain `cad-reference projected from Manufacturer Part Number`.
- Do not use fuzzy spelling correction. New aliases are added intentionally to the documented registry with fixtures.
- Apply global uniqueness validation only after projection, using the same case-insensitive comparison as the ODBC lookup.

#### ODBC identifier policy

- Treat table and column identity case-insensitively, matching the query layer.
- Reserve `Parts` for the merged projection. `orphanage` remains the documented fallback classification table and may not be redefined with conflicting semantics.
- Reserve `#` as the encoded classification separator. A source classification containing a literal `#`, or two classifications such as `a/b` and `a#b` that would map to one ODBC name, is a validation error with both source locations.
- Reject case-only table-name conflicts rather than silently merging differently spelled classifications.
- Map the complete documented Altium reserved-field set to its canonical public spelling; this includes the existing `description` -> `Description` and `value` -> `Value` behavior.
- For other fields, trim the source header but preserve its spelling. If two observed headers compare equal under ODBC rules but differ in spelling or normalize to the same reserved/public column, reject the conflict and report every source location.
- Order public columns deterministically by their case-insensitive canonical name with an ordinal tie-breaker. Schema union may add columns but cannot depend on filesystem iteration order.
- Preserve a conflicting original header/value in diagnostic provenance, not as a second indistinguishable ODBC column.

The default collision policy is strict:

- On initial load, any duplicate part key fails catalog construction and reports the normalized key plus every source location.
- On live refresh, a duplicate rejects the candidate snapshot and leaves the last known-good catalog active.
- There is no implicit first-wins, last-wins, folder priority, format priority, or exact-row deduplication. Those policies can silently link Altium to the wrong component.

#### Validation and diagnostics policy

The behavior contract must freeze this severity matrix before implementation:

| Condition | Initial load | Refresh | Diagnostic |
|---|---|---|---|
| Unsupported extension, Excel temporary file, or non-regular entry | Ignore | Ignore | Debug/info when logging is enabled |
| Empty workbook or empty/hidden worksheet | Skip according to the documented worksheet rule | Same | Warning with source and worksheet |
| Malformed supported file, encrypted workbook, invalid header, invalid row, missing key, identifier collision, or resource-limit breach | Fail catalog construction | Reject candidate; retain last known-good snapshot | Error with source, worksheet/row when available, and corrective action |
| Duplicate projected key | Fail catalog construction | Reject candidate; retain last known-good snapshot | Error listing the key and every provenance location |
| Transient sharing/partial-write failure | Bounded retry after debounce, then fail if still invalid | Bounded retry, then retain last known-good snapshot | Warning for retries, error on final rejection |

Do not publish a silently partial catalog after a supported source has a data error. Initial-load failures surface through the ODBC connection error and structured log. The catalog inspection CLI and DbLib builder print the same structured diagnostics for preflight use. Logs must never include unrelated cell contents or secrets beyond the field/key values needed to locate the error.

### Cache and file access

- Readers close every source handle after building a snapshot; Excel or another editor must be able to reopen, rename, and save a source after load.
- The driver serves queries from memory. It does not reread a spreadsheet for every ODBC operation.
- Publish snapshots through shared immutable ownership: each metadata/query operation captures one `shared_ptr<const CatalogSnapshot>` (or equivalent), and a successful refresh performs one atomic exchange after the candidate is complete.
- Eliminate the startup lost-change window by starting the watcher before the initial scan and reconciling after the scan, or by an equivalently proven sequence.
- A supported-file notification is debounced and rebuilds a complete candidate catalog outside the query lock. Changes arriving during a build set a dirty generation and schedule another rebuild.
- Notification-buffer overflow, watcher failure/restart, root removal/recreation, or any uncertain event sequence triggers a full reconciliation scan.
- A malformed or temporarily half-written file produces a diagnostic and does not destroy the last known-good snapshot. The error/retry policy is finalized in the behavior contract.
- The expected source files are small (approximately 800 rows or fewer), so correctness and simple full-snapshot refresh take priority over premature incremental-update complexity.

### Parser safety boundary

- Before selecting or integrating a spreadsheet library, freeze explicit limits for source bytes, worksheets, rows, columns, total cells, cell-string length, XML nesting, XLSX compressed-to-expanded size, and total expanded bytes. The approximately 800-row operating target informs conservative defaults but is not itself an enforceable safety boundary.
- Reject encrypted or password-protected workbooks. Do not execute macros, evaluate formulas, fetch external links, resolve network resources, or load embedded objects. Formula handling is limited to the documented cached-result policy.
- Apply the same bounds before allocating the full catalog snapshot, and include malformed-file, decompression-bomb, excessive-dimension, fuzz-corpus, and sanitizer coverage in each native reader's acceptance gate.

## Delivery sequence

### 1. Characterize current behavior, then freeze the contracts

- Treat every current-behavior statement in this plan as a hypothesis until it has a fixture or direct code/runtime evidence. Record the characterization evidence before approving new contracts.
- Inventory representative existing JSON catalogs for missing keys, duplicate keys, whitespace, field-casing differences, schema drift, and later-version-only fields so the intentional validation and schema-union changes have a measured migration impact.
- Characterize current ODBC table metadata, first-record schema discovery, `Parts`, classification tables, query filtering and literal escaping, UUIDv7 latest-version selection, ANSI sanitization, connection strings, error reporting, and live JSON refresh.
- Characterize placement and update lookup with the same logical part exposed in both `Parts` and its classification table, because the current generated DbLib enables both views with the same key predicate.
- Include fixtures showing that the current native driver and Python DbLib builder scan only the root even though the Windows watcher already requests subtree notifications. This mismatch is the central issue #4 regression case.
- Using that evidence, add proposed ADRs for automatic mixed-format discovery and global part identity, plus a proposed design document for discovery, reader dispatch, normalized catalog construction, refresh, diagnostics, and ODBC boundaries. Mark them accepted/current only after implementation and signoff agree with them.
- Add requirements and traceability for global part-key integrity, recursive discovery, content-only classification, existing JSON compatibility, multi-row tabular sources, unlocked files, cached queries, and deterministic refresh.
- Freeze exact scalar conversion, worksheet eligibility, identifier, key transport, resource limit, validation, and diagnostic contracts before selecting implementation APIs.
- Define the compatibility envelope explicitly: preserve the existing JSON document shape, latest-schematic-version selection, field values, classification, connection strings, and supported query behavior; intentionally replace first-record schema discovery with deterministic schema union and add projected-key validation. Publish migration diagnostics and fixture results for those intentional differences.

### 2. Adopt the normalized catalog and migrate JSON

- Replace the JSON-specific data-source boundary with a format-neutral catalog service before changing discovery behavior.
- Define a narrow reader interface: supported extensions in, one source path in, source records and diagnostics out.
- Make the current JSON parser the first reader adapter. It preserves the existing one-part-per-file and latest-schematic-version behavior and emits one source record rather than constructing ODBC tables itself.
- Move table creation, Altium field-name normalization, schema union, provenance, deterministic ordering, and immutable snapshot publication into the catalog layer.
- Keep SQL parsing, filtering, ODBC handles, and result binding outside the catalog and source readers.
- Give the catalog core and JSON adapter direct unit tests that do not require driver installation or a live ODBC connection.
- Build the catalog core as a reusable native library consumed by both the driver and a small catalog-inspection CLI. The CLI emits stable machine-readable table metadata and structured diagnostics; the Python DbLib builder invokes it instead of reimplementing discovery or consuming a manually maintained manifest.
- Prove the preserved root-level JSON invariants before adding recursion: document shape, latest-schematic-version selection, values, classification, connection strings, and supported query results.
- Add explicit approval tests for the intentional differences: deterministic schema union and failures caused by missing, ambiguous, or duplicate projected keys.

The clean boundary is:

`discovery -> source reader -> record normalizer -> catalog snapshot -> ODBC query layer`

All later source types enter through the same reader interface; none may write directly into ODBC-specific table state.

The design should converge on an interface shaped like the following without treating these provisional names as frozen API:

```text
SourceLocation
  root-relative path
  optional worksheet
  optional row

SourceRecord
  source field-name -> text/null values
  SourceLocation provenance

NormalizedRecord
  ordered field-name -> text/null values
  canonical cad-reference plus key-source provenance
  SourceLocation provenance

ReadResult
  zero or more SourceRecord values
  zero or more structured diagnostics

ISourceReader
  supported extensions
  read(source path) -> ReadResult

RecordNormalizer
  normalize(SourceRecord, alias registry) -> NormalizedRecord or diagnostics

CatalogBuilder
  build(all NormalizedRecord values) -> immutable CatalogSnapshot
```

Readers parse source files and return fields plus provenance only. The shared `RecordNormalizer` owns header normalization, alias matching, and canonical key projection. `CatalogBuilder` owns key validation, classification routing, `orphanage`, Altium column-name mapping, schema union, ordering, duplicate diagnostics, and table/`Parts` construction. The ODBC layer receives an immutable `CatalogSnapshot` and owns only metadata and query behavior.

### 3. Deliver recursive JSON discovery through the new boundary

- Introduce one cycle-safe recursive discovery function with stable relative paths and deterministic sorting.
- Feed its JSON results through the JSON reader into the normalized catalog for initial load and every full reload.
- Treat the watcher callback path as root-relative, not merely a basename, so identical filenames in different subdirectories cannot collide.
- Update the DbLib classification scan to recurse with matching rules.
- Add nested add, modify, delete, rename, duplicate-basename, uppercase-extension, ignored-file, and directory-only-change tests.
- Keep classification entirely content-driven and document the public issue behavior.

Review this milestone independently, but do not treat it as a separate release gate; release readiness is decided only by this plan's focused signoff and closeout sequence.

### 4. Define and implement tabular readers

#### Golden cross-format corpus

- Before implementing a tabular reader, define one small text-only canonical component dataset and one expected normalized-catalog manifest containing tables, ordered columns, rows, projected keys, and classification results. Text-only values form the portable equivalence subset shared by JSON, CSV, XLSX, and XLS.
- Represent that logical dataset as the existing one-part-per-file JSON format, CSV, XLSX, and XLS. Each reader must pass the common corpus independently before it can participate in mixed-source tests.
- Add format-specific conversion fixtures for missing versus blank values, integers, decimals, booleans, dates, Unicode, formula cached results and absent caches, CSV encoding and quoting, spreadsheet error cells, and unsupported JSON nested values. These fixtures validate the frozen conversion contract; they are not required to normalize unlike native types into accidental equivalence.
- Run a mixed directory containing all formats with deliberately distinct keys and compare it with the corresponding combined manifest.
- Add negative companion fixtures for missing keys, alias ambiguity, duplicate keys across formats, malformed source files, resource-limit violations, and partial writes.
- Keep catalog/reader parity tests independent of ODBC registration and Altium so they run quickly and deterministically. Reuse the corpus for a smaller ODBC metadata/query suite and an installed-driver smoke test.
- Treat committed XLSX/XLS fixtures as governed test artifacts: record their source dataset, generation procedure, immutable hashes, and licenses/tools used to produce them.

#### CSV

- Implement the documented read-only CSV contract with RFC 4180 quoting behavior, CRLF/LF handling, UTF-8 BOM handling, embedded delimiters/newlines, and deterministic diagnostics.
- Avoid adding a third-party dependency unless the contract cannot be satisfied cleanly by a small, tested parser.
- Pass the common golden corpus, CSV conversion fixtures, negative fixtures, and resource limits before enabling `.csv` discovery by default.

#### XLSX

- Run a focused MSVC/CMake static-link spike before vendoring.
- Prefer [xlsxio](https://github.com/brechtsanders/xlsxio) for the spike because its read API is designed for spreadsheet-shaped data, is streaming, and is MIT-licensed. Compare it with [OpenXLSX](https://github.com/troldal/OpenXLSX) if xlsxio's Expat/minizip/zlib dependency chain or MSVC integration is too costly.
- Selection gates are: read-only worksheet enumeration, Unicode paths and cells, cached formula values, no persistent file handle, x64 MSVC/Ninja build, static packaging into the driver, dependency/license footprint, malformed-input behavior, and compatibility with the 800-row use case.
- Record the choice and rejected alternative in an ADR before integrating it.
- Pass the common golden corpus, XLSX conversion fixtures, negative fixtures, and resource limits before enabling `.xlsx` discovery by default.

#### XLS

- Use [libxls](https://github.com/libxls/libxls) 1.5 or newer as the selected legacy `.xls` reader, subject to the same pinning, licensing, MSVC, malformed-input, and static-packaging checks.
- Prefer its buffer/error-return APIs so parsing does not retain an open workbook file and malformed inputs cannot terminate the host process.
- Treat `.xls` and `.xlsx` as distinct formats with distinct readers; libxls does not provide `.xlsx` support.
- Pass the common golden corpus, XLS conversion fixtures, negative fixtures, and resource limits before enabling `.xls` discovery by default.

### 5. Integrate mixed sources and refresh

- Register JSON, CSV, XLSX, and XLS readers under the same recursive discovery pass.
- Build tables and the `Parts` view from the merged normalized records without source precedence or provider configuration.
- Exercise mixed classifications, schema union, repeated part keys, same filenames in different folders, empty sources, malformed sources, and simultaneous source changes.
- Test every documented alias and separator/case variant, blank fallback, conflicting same-group aliases, missing keys, and duplicate keys within a source, across files, across formats, across classification tables, and with case-only differences. Verify initial-load failure and last-known-good refresh behavior.
- Expand watcher filtering to all supported extensions and coalesce noisy editor save sequences.
- Verify query threads retain the last immutable snapshot while the next snapshot is built and swapped.
- Exercise notification-buffer overflow, watcher failure and restart, source-root removal and recreation, and changes arriving during a build. Every uncertain sequence must reconcile with a full scan, and an in-flight query must continue safely against its captured old snapshot.
- Confirm with process-level tests that files are not locked between refreshes.

### 6. Keep helper tooling and documentation aligned

- Make `odbc-monkey-dblib` invoke the native catalog-inspection CLI and consume its stable machine-readable table metadata and diagnostics. Both tools therefore execute the same catalog library; do not add four Python parsers or a separately maintained manifest.
- Always generate DbLib lookup predicates against canonical `cad-reference`. Preserve the existing default option spelling for compatibility, and produce the documented migration error for a non-default `--primary-key`; the helper and driver's validation key cannot diverge.
- Update setup, architecture, design, contracts, requirements, test strategy, vendor inventory, artifact governance, JSON-format compatibility notes, examples, and release notes.
- Publish the key-projection priority as one compact table in user documentation and include JSON, CSV, and spreadsheet examples showing explicit `cad-reference`, IPN fallback, and MPN fallback. Diagnostics and validation output must use the same terminology.
- Add one mixed-source example with nested folders that proves folders do not classify records and that spreadsheets can contain multiple parts while JSON remains one part per file.
- Document supported spreadsheet features and explicit non-features; this is a component-table reader, not a general Excel calculation engine.
- Run the full local `dev-std` audit, native tests, Python tests, installed-driver smoke tests, and DbLib/Altium compatibility checks.

## Dependency governance

Every selected third-party component and transitive library must have:

- an exact immutable version or commit;
- upstream URL, license, copyright notice, and local patch record;
- a reproducible update procedure and CMake integration that does not download code during a normal build;
- entries in the repository vendor and artifact inventories;
- focused tests for malformed input and the APIs on which the driver relies;
- documented parser bounds and regression inputs for excessive dimensions, archive expansion, invalid encodings, and unsupported workbook features;
- a documented security/update owner and replacement strategy.

The current vendored `nlohmann/json.hpp` is brought under the same governance while this boundary is being changed.

## Explicitly out of scope

- Using directory names as table names, classification, or any other semantic field.
- Requiring users to choose exactly one provider or file type.
- Writing back to JSON, CSV, XLSX, or XLS files.
- Evaluating Excel formulas, preserving workbook formatting, charts, macros, or images.
- Changing the JSON document format, adding unversioned or multi-part JSON shapes, or treating entries in one `versions` array as different parts. The array contains schematic versions for one logical part.
- Replacing ODBC, redesigning the query language, or broadening SQL support unrelated to catalog ingestion.
- CI implementation or adoption of pull request #2; this plan uses the local signoff established by `dev-std-adoption`.
- Unbounded performance work for very large workbooks. The accepted operating target is small component tables, approximately 800 rows per source file.

## Closeout evidence

- A requirements-to-tests matrix covers every exit criterion and every supported extension.
- Before/after compatibility fixtures demonstrate the preserved root-level JSON invariants and explicitly enumerate approved schema-union and key-validation differences.
- Recursive discovery and refresh tests prove issue #4, including nested changes after connection.
- Mixed-source fixtures prove schema union, classification, table metadata, queries, and `Parts` behavior.
- File-handle tests prove sources can be renamed or rewritten after load.
- Vendor records and reproducible-build checks cover direct and transitive dependencies.
- Durable findings are distilled into governed docs before this plan and its logs are removed through the standard closeout process.
