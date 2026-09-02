# Writing a Business Context Document for Your Digital Twin

## Why this matters more than any other input

The extractor tool can pull out *what* your model contains — table names, column types, DAX formulas, relationships, visual field bindings. What it cannot know is *why* any of it looks the way it does: why a measure ignores certain filters, what "Round 2" means to your business, or which table is the safe one to join on. That's the gap a business context document fills, and it's the single biggest lever on how useful your digital twin is. Two models with identical schemas can get wildly different quality of help from an LLM depending on how well the context is written.

This file becomes the **"## Business Context"** section at the top of `llm_ready_input.md` — the LLM reads it before it reads any table or DAX, so it frames everything that follows.

## Principles that make a context document actually useful

1. **State the *why*, not just the *what*.** "Baseline Cost is calculated by joining allocation to contracted rates" is what. "Baseline Cost exists to answer 'what would we be paying today if the current contract allocation stayed in place' — it's the benchmark every other cost gets compared against" is why. The LLM needs the why to extend your logic correctly instead of guessing.
2. **Define the grain of every important table.** "One row per Contractor + Rate Code + Division" prevents an LLM from writing a measure that silently double-counts because it assumed a different grain.
3. **Write your glossary in both directions.** Business people say "Prime"; the model might call it `Current Prime` or `Contractor Code`. Spell out the mapping — this is what lets someone non-technical ask a question in their own words and get a correct measure back.
4. **Call out anything that looks like a bug but isn't**, and anything that IS a known issue you haven't fixed yet. Without this, the LLM will either "fix" intentional behavior or repeat known mistakes.
5. **Tell it what role you want it to play.** "Help me write new measures matching existing style," "audit this model for performance," "propose new visuals" are different jobs — say which ones you actually want, and any conventions (naming, formatting, patterns to avoid) it should follow.
6. **Be specific over exhaustive.** A tight, opinionated paragraph per table beats a bland one-line description of every column — the tool already extracts column names and types; your job here is meaning and intent, not restating the schema.

## Template

Copy the block below into a new `.txt` file, fill in every `[bracketed]` placeholder, delete sections that don't apply, and select that file in the tool's "Business context" field. Keep the `###` headers — they render cleanly inside the generated dossier.

```
### USER-DEFINED OPERATIONAL ENVIRONMENT
- **Business/Sourcing Domain:** [One line — what business area is this model for?]
- **Project Scope:** [What decisions does this model support? Who acts on its output, and what does "success" look like? Mention any related tools/processes it feeds into or draws from.]
- **Primary Data Sources:** [Where does the data physically come from? How does it get updated — manual drop, automated feed, gateway? Any quirks in how source files are named/organized?]
- **Target Audience:** [Who looks at the reports, and what's their technical level? Executives skimming a summary vs. analysts drilling into detail need different things from a suggested visual or measure.]
- **Original Ask / Problem Statement:** [Paste the original brief or business question this model was built to answer, in the requester's own words if you have it. This is often the richest single source of intent.]

### DATA MODEL AND SCHEMA
- **Schema shape:** [Star schema / snowflake / other. Name the core dimensions and fact tables at a high level.]
- **Table: [TableName]** — [What does one row represent, in business terms? What's the grain / unique key? What is it used for? Anything about it that's non-obvious?]
- **Table: [TableName]** — [Repeat for every table where the name alone doesn't make the meaning obvious. You don't need this for every dimension table, but do it for every fact table.]

### SYSTEM ARCHITECTURE & GATEWAYS
- **Gateway / Deployment:** [Power BI Service, Report Server, Desktop-only, etc.]
- **Refresh Cadence:** [Scheduled / manual / real-time, and how often]
- **Known Performance Notes:** [Slow refresh, slow visuals, anything relevant if you want the LLM's help optimizing]

### KEY CALCULATIONS & BUSINESS LOGIC
[For your most important measures, explain the business intent in plain English — the raw DAX is already extracted separately, so focus on *why* the logic is shaped the way it is, any filter context tricks (ALL, ignored filters) and why they're there, and how it should behave when extended.]
- **[Measure Name]:** [What business question does this answer? What would break if someone changed its filter logic without understanding why it's there?]

### KNOWN ISSUES & GOTCHAS
- [Anything that looks wrong but is intentional]
- [Anything that IS wrong that you haven't fixed yet, so the LLM doesn't propagate it]
- [Data quality quirks: duplicates, nulls, manual overrides, known bad source rows]

### TERMINOLOGY GLOSSARY
- **[Business term]** = [field/table it maps to, and why the name differs]
- **[Business term]** = [field/table it maps to]

### WHAT YOU WANT FROM THE DIGITAL TWIN
- [List the specific jobs you want it to help with — e.g. writing new DAX measures, proposing new visuals/pages, explaining relationships to new team members, auditing performance, reviewing calculation logic for correctness]
- **Conventions to follow:** [Naming style, formatting preferences, patterns to prefer or avoid — e.g. "always use variables in DAX," "match Title Case measure names," "avoid bidirectional relationships unless there's a stated reason"]

### OPEN QUESTIONS / OBJECTIVES
[Anything you're actively trying to figure out — this is a great place to restate the actual questions you plan to ask the digital twin first, so its first answers are already aimed at what you need.]
```

## A worked example

The context file already in `config/analytical_context.txt` in this project is a real, filled-in example from an RFP pricing/sourcing model — it's a good reference for the level of specificity to aim for, particularly the "Original Ask" and "Key Calculations" sections.
