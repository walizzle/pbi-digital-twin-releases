# Changelog

All notable changes to this project are documented here. Versioning follows [Semantic Versioning](https://semver.org/) (MAJOR.MINOR.PATCH). Each version from 1.3.1 onward has a matching git tag.

## [1.7.0] - 2026-09-01
### Added
- Cloud extraction is now wired into the GUI and the shipped `.exe`: `orch2_gui.py` has a "Cloud Model (Power BI Service)" tab alongside the existing "Local File (.pbix)" tab, both sharing the log pane and Open Dossier/Open Output Folder controls. Signing in shows the device code prominently in the window (with Copy Code / Open Sign-in Page buttons) instead of requiring the user to find it in a scrolling log, and auto-opens the sign-in page.
- `device_code_login()` and `run_cloud_pipeline()` in `orch2_cloud.py` now accept an `on_device_code` callback fired right after the code is issued, for exactly this GUI use case; CLI behavior (printing the code) is unchanged.
- `build_kit/requirements.txt` now includes `msal`/`requests` and `build.bat` passes `--collect-all msal` to PyInstaller, so the single built `.exe` supports both local and cloud extraction out of the box - no separate download or install step for cloud use.
### Note
Verified with an offline GUI smoke test (widget construction, device-code display/clipboard/browser-open, run-button mutual exclusion, error/complete state transitions) under a virtual display. The actual Windows `.exe` build via PyInstaller was not run from this environment (Linux sandbox, no way to produce/test a Windows binary) - if `msal` needs additional PyInstaller hooks beyond `--collect-all msal`, that will surface as a runtime import error in the built exe's console/log.

## [1.6.0] - 2026-09-01
### Added
- `orch2_cloud.py` — extracts a dossier (schema, DAX measures, relationships, real row counts, measured refresh-history timing) directly from a Power BI Service (cloud) semantic model on Pro/shared capacity, with no local `.pbix`, no Azure AD app registration, and no Premium/XMLA endpoint required. Authenticates via OAuth2 device-code flow reusing the same public client ID Microsoft's own `Connect-PowerBIServiceAccount` PowerShell cmdlet uses. Schema (tables/columns/measures/relationships) comes from DAX `INFO.VIEW.*` functions run through the `executeQueries` REST endpoint; row counts come from `COUNTROWS()` the same way; refresh timing comes from the refresh history REST API, with per-table detail when available via the Enhanced Refresh API. Does not retrieve Power Query M code, VertiPaq compressed sizes, or report visuals — see `Cloud_Model_Guide.md`.
- `Cloud_Model_Guide.md` — documents three ways to get a dossier from a cloud-hosted model: downloading the `.pbix` and using the existing tool, a Desktop live-connect + DAX Studio workflow for real table sizes, and `orch2_cloud.py` for everything else.
- `requirements-cloud.txt` — the `msal`/`requests` dependencies needed only for `orch2_cloud.py`; the local, offline tool gains no new dependencies.

## [1.5.1] - 2026-08-14
### Fixed
- Power Query functions and other non-loaded queries saved as their own individual TMDL script tab (rather than inside the whole-model script) were silently dropped: `run_pipeline` only ever parsed the single TMDL file matched by `find_model_tmdl()`, so `expression` blocks living in any other extracted `.tmdl` file never made it into the "Non-Loaded Power Query Expressions" section. All extracted `.tmdl` files are now scanned for `expression` blocks and merged in (deduplicated by name).
- Excel/PBIX M-code extraction (`extract_excel_m_code`, `extract_pbix_m_code`, `extract_m_code_from_mashup`) silently swallowed every failure path (malformed customXml, missing `DataMashup` part, missing `Formulas/Section1.m`, base64/zip errors), so an empty `m_queries.txt` gave no indication of why. Each failure path now prints a specific reason to the log, and the `Formulas/Section1.m` lookup is now case/slash-insensitive instead of an exact-string match.

## [1.5.0] - 2026-08-14
### Added
- Static, offline signals for diagnosing slow model refreshes: Excel sheet/table used-range dimensions are now flagged when unusually large, including a specific call-out for used ranges near Excel's row limit (almost always leftover formatting, not real data).
- PBIX file size and a breakdown of its largest internal zip parts.
- Power Query M-code performance heuristics (`analyze_m_performance`) scanning each table/expression's M code for common refresh-time anti-patterns: long transformation chains on non-folding sources (Excel/CSV/Web), stacked merges/joins, `Value.NativeQuery` with folding explicitly disabled, Web/API calls (flagged as likely per-row when near an `each`), and tables referenced by multiple other queries without `Table.Buffer`. Cross-references M code against Excel range sizes so a table's own oversized source can be named directly.
- New dossier sections: "PBIX File Size" and "Potential Refresh-Time Bottlenecks (Static Analysis)", plus corresponding Executive Summary counts. All of this is heuristic/regex-based (no live connection to Power BI Desktop or Analysis Services) — findings are leads to investigate, not verified facts; see the note at the top of each new section.

## [1.4.0] - 2026-08-06
### Added
- Each run now writes into its own timestamped subfolder (`output_dir/YYYYMMDD_HHMMSS/`) instead of overwriting the previous run's files. The dossier file itself is also named with the timestamp (`llm_ready_input_YYYYMMDD_HHMMSS.md`), since it's the file most often shared on its own by email.
### Changed
- `run_pipeline()` no longer takes a `dossier_output` parameter — it computes the run folder and all file paths internally and returns them as a dict (`run_dir`, `dossier`, `timestamp`). The GUI and CLI were updated to use the returned paths instead of pre-computing them.

## [1.3.2] - 2026-08-05
### Added
- `__version__` in `orch2.py`, shown in the GUI window title, as the single source of truth for the app's version going forward.
- This changelog.

## [1.3.1] - 2026-08-05
### Added
- Latest built executable (`PBI_Digital_Twin_Extractor.exe`) committed to the repo, tracked via a targeted `.gitignore` exception (all other build-generated `.exe` files stay ignored).

## [1.3.0] - 2026-08-05
### Added
- Non-loaded Power Query expression extraction — TMDL `expression` blocks (staging queries, or queries with "Enable Load" disabled) are now parsed. Previously these never appeared as a table/partition and were silently dropped entirely. Recovered 28 such queries from the reference model, including live-SharePoint staging queries feeding the loaded tables.
- Multiple Excel file support — the GUI has an Add/Remove file list instead of a single picker; the pipeline extracts M-code and structure per workbook, labeled by filename.
- Excel workbook structure extraction — sheet names/dimensions, workbook-level named ranges, and Excel Tables (name + range), via a fast openpyxl read-only pass. Deliberately excludes formula scanning and pivot tables to stay fast even on large workbooks (~0.7s on a 108K-row sheet).
- New dossier sections: "Excel Workbook Structure" and "Power Query Expressions Not Loaded to the Model", plus corresponding Executive Summary counts.
### Fixed
- A `queryGroup:` TMDL property line was leaking into extracted DAX/M expression bodies instead of being recognized as metadata.
- Same-line expression assignments (e.g. `measure X = let ...`) corrupted the indentation of every following line when dedenting; only the bare-trailing-`=` form (expression starting on the next line) worked correctly before. Affected measures, calculated columns, and table partition M code.
- `DataMashup` binary parsing sliced to the end of the buffer instead of using the format's own length prefix, which silently returned an empty archive whenever trailing Permissions/Metadata sections were present — this was masking the fact that Excel M-code extraction wasn't working at all.

## [1.2.0] - pre-repository
### Added
- `README.md`, `TMDL_Extraction_Guide.md`, and `Business_Context_Guide.md`.
- `build_kit/` — a source-only distributable (`orch2.py`, `orch2_gui.py`, `requirements.txt`, `build.bat`, instructions) so the executable can be built locally by someone with just Python and VS Code, since `.exe`/`.zip` attachments are commonly blocked by mail filters.

## [1.1.0] - pre-repository
### Changed
- The Excel workbook is now optional in both the GUI and the pipeline; previously required.

## [1.0.0] - pre-repository
### Added
- Initial rewrite of the extraction pipeline: correct UTF-16LE decoding of `DataMashup`/`.tmdl` parts (previously silently corrupted), a TMDL model parser (tables/columns/measures/relationships) used in place of the legacy `DataModelSchema` JSON (empty on newer PBIX files), and PBIR-format report/visual extraction (pages, visual types, field bindings) replacing the old single-file `Report/Layout` approach.
- `orch2_gui.py` — a Tkinter file-picker GUI wrapping the pipeline.
- First packaged executable (`PBI_Digital_Twin_Extractor.exe`) via PyInstaller.

---
Versions 1.0.0–1.2.0 predate this git repository (that work was squashed into the initial commit, so no per-version tag exists for them); 1.3.0 is tagged at the initial commit itself, and 1.3.1 onward are tagged at their own commits.
