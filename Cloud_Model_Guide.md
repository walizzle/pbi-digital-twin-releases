# Working with Power BI Service (Cloud) Models

`orch2.py` only reads a local `.pbix` file. If your model lives in the Power BI Service and you don't have a local copy, use one of these three approaches, roughly in order of preference:

## Tier 1 — Download the `.pbix` and use the normal tool
If your workspace permissions allow it: **File > Download this file** (or the "..." menu on the dataset/report) in the Power BI Service. If that works, you have a normal `.pbix` — run `orch2.py`/the GUI exactly as documented in `README.md`. This gets you everything, including M code and report visuals.

## Tier 2 — Desktop live-connect + DAX Studio (for real table sizes)
If download is disabled but you have Power BI Desktop: **Get Data > Power BI semantic models**, pick the cloud model, and build a report against it (a "live connection" — no local copy of the data model, just a session). Then open **DAX Studio**, connect to that running Desktop session, and use its **Vertipaq Analyzer** view for real row counts and compressed table/column sizes — the same numbers XMLA would give you, without needing Premium capacity, since DAX Studio talks to Desktop's local session either way. This doesn't produce a dossier file; it's a manual diagnostic pass. M code still isn't visible this way (live-connect reports don't have local Power Query at all — the M lives in the cloud dataset).

## Tier 3 — `orch2_cloud.py` (schema + real row counts + real refresh timing, no download, no Premium)
For a Pro/shared-capacity workspace where you can't download the file: `orch2_cloud.py` pulls schema, measures, relationships, real row counts, and measured refresh-history timing straight from the Power BI Service, and writes a dossier in the same style as the local tool.

**From the GUI (`PBI_Digital_Twin_Extractor.exe`):** open the **"Cloud Model (Power BI Service)"** tab, enter the workspace and dataset (name or ID), pick an output folder, and click **Sign In & Extract**. A device code appears in the window (also copied to the log) with a "Copy Code" and "Open Sign-in Page" button — the sign-in page opens automatically. Everything else runs the same as the Local tab: progress in the log, then "Open Dossier" / "Open Output Folder" when it's done.

**From the command line:**
```
pip install -r requirements-cloud.txt
python orch2_cloud.py --workspace "My Workspace" --dataset "My Model" --output ./cloud_output
```
It prints a short device code and URL — sign in with a browser (any device) using an account that has at least Viewer access to the workspace. No Azure AD app registration is needed: it reuses the same Microsoft-published client ID that Microsoft's own `Connect-PowerBIServiceAccount` PowerShell cmdlet uses.

**Important — tell your security team first.** This is still a real OAuth sign-in against your tenant, using a Microsoft first-party app. If your org restricts consent to admin-approved apps only, it will fail with an admin-consent error (no client ID avoids that — only a tenant admin can grant it). And even when it works, your tenant's sign-in logs will show a device-code login attributed to Microsoft's Power BI management app, which can look surprising out of context to a SOC even though nothing new was registered. Get sign-off before pointing this at a client tenant.

**What you get:** tables/columns/measures (with full DAX)/relationships via DAX `INFO.VIEW.*` functions, real `COUNTROWS()` per table, and refresh history with measured (not estimated) durations — plus a per-table timing breakdown when the refresh was run via the Enhanced Refresh API.

**What you don't get without Tier 1 or 2:** Power Query M code, compressed/VertiPaq sizes, and report visual field bindings — none of these are exposed through the Pro-tier REST/DAX surface this uses.

If a DAX query fails with a compile error naming a specific column (a few of the measure/relationship field names couldn't be independently verified against Microsoft's docs from this environment), the error message names the exact bad reference — it's a one-line fix in `orch2_cloud.py`.
