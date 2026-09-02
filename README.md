# Power BI Digital Twin Extractor — Downloads

This repo holds the built `.exe` and documentation for the team. Source development happens in a separate private repo — ping the maintainer for access if you need that.

**⬇️ [Download PBI_Digital_Twin_Extractor.exe](https://raw.githubusercontent.com/walizzle/pbi-digital-twin-releases/main/PBI_Digital_Twin_Extractor.exe)** (right-click → Save Link As, or click then use your browser's download; some browsers preview it inline instead of downloading — if so, right-click the link.)

This repo updates automatically whenever the source changes, so it's always current — no need to ask for a fresh copy.

---

# Power BI Digital Twin Extractor

This tool reads a Power BI file (`.pbix`) and, optionally, its source Excel workbook, and produces a single text file (`llm_ready_input.md`) containing everything an LLM needs to act as a "digital twin" of your model: every table and column, every DAX measure, every relationship, every report page and the fields each visual uses, the Power Query (M) transformation logic, and static signals for spotting refresh-time bottlenecks (oversized Excel source ranges, long non-folding M transformation chains, stacked merges, per-row web calls, and more).

You then hand that file to an AI assistant (Claude, ChatGPT, Copilot, etc.) as context, and ask it to help you write new measures, design new visuals, explain relationships, debug slow DAX, or review the model for issues — without it ever needing direct access to your Power BI file.

---

## 1. What you need before you start

| Item | Where to get it |
|---|---|
| `PBI_Digital_Twin_Extractor.exe` | The download link at the top of this page |
| Your `.pbix` file | Required |
| Your source Excel workbook (`.xlsx`) | Optional — only needed if you want the Power Query steps that live in Excel included |
| A business context file (`.txt`) | Optional but **strongly recommended** — see [Business_Context_Guide.md](Business_Context_Guide.md) |

**Important — do this first:** for the tool to pull out your tables, columns, measures, and relationships, your `.pbix` file needs to have a full-model TMDL script saved inside it. This is a one-time (and then occasional) manual step in Power BI Desktop. If you skip it, the tool will still run, but the Executive Summary in the output will show `Tables: 0`, `Measures: 0`, etc.

👉 **Follow [TMDL_Extraction_Guide.md](TMDL_Extraction_Guide.md) before running the extractor for the first time**, and again any time you've made significant changes to the model (new tables, measures, or relationships) and want the digital twin to reflect them.

**Don't have a local `.pbix`?** If your model only lives in the Power BI Service (cloud), use the **"Cloud Model (Power BI Service)"** tab in the same app — no Azure AD app registration or Premium capacity required, just a device-code sign-in. See [Cloud_Model_Guide.md](Cloud_Model_Guide.md) for what it can and can't retrieve compared to a local file, and two additional no-code workflows for the gaps.

---

## 2. Running the tool

1. Double-click `PBI_Digital_Twin_Extractor.exe`.
   - Windows may show a blue "Windows protected your PC" warning the first time, because the file isn't digitally signed. Click **More info → Run anyway**. This is expected and safe — it's just Windows being cautious about unsigned executables.
2. The window has two tabs — pick the one that matches where your model lives:
   - **Local File (.pbix)** — for a `.pbix` file on your machine:
     - **Power BI file (.pbix)** — click Browse and select your file. *(Required)*
     - **Excel file (.xlsx, optional)** — click Browse and select your source workbook if you want its Power Query logic included. Leave blank to skip.
     - **Business context (.txt, optional)** — click Browse and select your filled-in business context file. Leave blank if you haven't created one yet (see below).
     - **Output folder** — auto-filled next to your `.pbix`, or choose your own.
     - Click **Run Extraction**.
   - **Cloud Model (Power BI Service)** — for a model that only lives in the cloud (see [Cloud_Model_Guide.md](Cloud_Model_Guide.md) for what this can and can't retrieve):
     - Enter the **workspace** and **dataset** (name or ID) and an **output folder**.
     - Click **Sign In & Extract**. A device code appears in the window (also copied to the log); the sign-in page opens automatically — enter the code there.
3. Watch the log pane — for a large model this can take a minute or two. When it's done you'll see a confirmation popup and the final counts.
4. Click **Open Dossier** to view the result, or **Open Output Folder** to see everything that was generated.

---

## 3. What's in the output folder

| File | What it is | Do you need to look at it? |
|---|---|---|
| `llm_ready_input.md` | **The main deliverable.** One markdown file with your business context, every table/column, every relationship, every report page + visual + the fields it uses, every DAX measure, the Power Query M code, and a "Potential Refresh-Time Bottlenecks" section flagging things like oversized Excel ranges and un-folded/stacked M transformations. This is what you give to the LLM. | Yes — this is the file you'll actually use |
| `model_schema.json` | The same structured data as JSON, for anyone who wants to script against it | Only if you're building something on top of this |
| `dax_inventory.txt` | Just the DAX measures, one after another, in a plain-text file | Handy if you only want to review/audit measures |
| `m_queries.txt` | Just the Power Query M code | Handy for reviewing ETL logic in isolation |
| `tmdl/` | The raw `.tmdl` scripts pulled from your `.pbix`, decoded and readable | Reference only — `llm_ready_input.md` already includes this |

---

## 4. Using `llm_ready_input.md` with an LLM

1. Open a chat with your LLM of choice (Claude, ChatGPT, Copilot...).
2. Attach/upload `llm_ready_input.md`, or paste its contents in, as context at the start of the conversation.
3. Ask it questions like:
   - *"Write a DAX measure that calculates the % gap between Bid Cost and Baseline Cost, following the same style as the existing measures."*
   - *"I want a new page that compares this round's pricing to the last round by division — what fields and visual types would you use, based on what's already in the model?"*
   - *"Explain why FactRFPRates.Bidder has a bidirectional relationship to DimBidders — is that intentional or risky?"*
   - *"Review my 'Baseline Cost' measure for performance issues given the table sizes described in the business context."*
   - *"A stakeholder says the numbers on the Summary View page don't reconcile with Bid Explorer — where would you start looking?"*
   - *"My refresh takes 20 minutes — walk through the 'Potential Refresh-Time Bottlenecks' section and tell me which finding to tackle first."*

The more specific your business context file is (see below), the better the answers — an LLM that knows *why* a measure is shaped the way it is, or what "Round" and "Prime" mean in your business, gives dramatically better suggestions than one just looking at raw table/column names.

**A note on size:** for a large report (dozens of pages, hundreds of visuals), `llm_ready_input.md` can get large. If your LLM has a smaller context window, you can trim the file down to just the sections you need for a given question (e.g. just "Tables & Columns" + "DAX Measures" for a measure-writing task) rather than pasting the whole thing every time.

---

## 5. Keeping the digital twin up to date

Re-run the tool whenever the model or report changes meaningfully. Two things to remember:

- If you added/changed **tables, columns, measures, or relationships**, re-save the full-model TMDL script first (see the [TMDL guide](TMDL_Extraction_Guide.md)) — otherwise the tool will still be reading the old snapshot.
- If your **business context** changed (new phase of the project, new key calculations, new known issues), update your context `.txt` file too — see [Business_Context_Guide.md](Business_Context_Guide.md).

---

## 6. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Executive Summary shows `Tables: 0`, `Measures: 0` | No full-model TMDL script saved in the `.pbix` | Follow the [TMDL Extraction Guide](TMDL_Extraction_Guide.md), re-save the `.pbix`, and re-run |
| `Power Query M Code` section is empty | No `.xlsx` was selected, or the workbook has no Power Query queries | Select your source Excel file when running the tool |
| `Report Pages & Visuals` section is empty | Very old `.pbix` report format not recognized | Should be rare on current Power BI Desktop versions; let the tool owner know if you hit this |
| Windows SmartScreen blocks the `.exe` | It's unsigned | Click "More info" → "Run anyway" |
| The tool window doesn't respond while running | Normal for large models — extraction runs in the background but can take a minute or two for hundreds of visuals | Wait for the log to finish; don't close the window |
