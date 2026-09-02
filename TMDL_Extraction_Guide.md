# Extracting a Full-Model TMDL Script from Power BI Desktop

## Why this step exists

Modern Power BI files (`.pbix`) don't store your data model as readable text. They store it as a compressed Analysis Services backup (you'll sometimes see this called the "DataModel" stream) — great for Power BI itself, unreadable by a simple script or an LLM.

The workaround: Power BI Desktop can generate **TMDL** (Tabular Model Definition Language) — a plain-text description of your entire model: every table, column, measure, relationship, and the Power Query source behind each table. If you save that script inside your `.pbix`, the extractor tool can read it directly. If you skip this step, the tool has no readable source for your model and the Executive Summary will show zeros for Tables/Measures/Relationships.

You only need to redo this when your **model** changes (new tables, columns, measures, relationships). Report-only changes (new pages, new visuals) don't require it — those are read directly from the report definition.

## Steps (Power BI Desktop)

> Menu wording can shift slightly between Power BI Desktop versions. If a label below doesn't match exactly, look for the closest equivalent — the underlying feature has been available since late 2024.

1. **Open your `.pbix` in Power BI Desktop.**

2. **Enable TMDL view, if it's not already on:**
   - File → Options and settings → **Options** → **Preview features**
   - Check **"TMDL view"** (sometimes listed under Model view options)
   - Restart Power BI Desktop if prompted.

3. **Switch to Model view** (the icon on the left rail that shows table relationships).

4. **Open the TMDL script pane:**
   - Look for a **TMDL view** toggle/button in the Model view ribbon or in the bottom-right of the window.
   - This opens a script pane alongside your model diagram.

5. **Generate a script for the whole model:**
   - In the TMDL explorer/tree on the left side of the script pane, select the **top-level "Model" node** (not an individual table).
   - Right-click it and choose the option to open/copy its TMDL definition (wording varies: "Copy definition," "New script," or it may open automatically when you select Model).
   - You should see a script starting with:
     ```
     createOrReplace

         model Model
             culture: en-US
             ...
             table YourTableName
                 ...
     ```
     If it only shows one table instead of `model Model` at the top, you've selected a single table's script instead of the whole model — go back and select the Model node itself.

6. **Save the script as a named tab.** The script pane supports multiple saved script tabs (you may see existing ones like "Script 1"). Give this one a clear name if possible, or just make sure it's saved before moving on.

7. **Save the `.pbix` file itself** (Ctrl+S). Saved TMDL scripts are embedded inside the `.pbix` under a `TMDLScripts/` folder — the extractor tool looks for them there.

## Verifying it worked

Run the extractor tool against the updated `.pbix`. In the log output (or console, if running the Python script directly) you should see a line like:

```
Model parsed from TMDL script: TMDLScripts/Script%202.tmdl
```

And the Executive Summary at the top of `llm_ready_input.md` should show non-zero counts for Tables, Columns, Measures, and Relationships.

If instead you see:

```
No full-model TMDL script found; used legacy DataModelSchema (may be empty for newer PBIX files).
```

...it means none of the saved TMDL scripts in the file contain a full `model Model` definition — only individual table scripts, calculation-group scripts, or nothing at all. Go back to step 5 and make sure you selected the top-level Model node, not a single table.

## A note on repeatability

This is a manual step because Power BI Desktop doesn't auto-export TMDL on every save. If re-doing this before each extraction becomes a hassle for your team, the alternative is a fully automatic approach that connects directly to the local Analysis Services instance Power BI Desktop runs while your file is open (via ADOMD.NET/TOM), avoiding the manual export entirely — more setup complexity, but zero manual steps going forward. Worth considering if this becomes a frequent workflow.
