---
name: qlik-load-script
description: "Write, complete, or extend Qlik Sense load scripts (.qvs) for data preparation. Primary use case is preparing flat datasets for ML experiments with Qlik Predict — engineering features, handling train/test splits, and outputting QVD files. Also covers general Qlik load script authoring: completing scripts from starting code, comments, or prompts. Use this skill whenever the user mentions Qlik load script, .qvs files, Qlik data load, QVD output, data preparation for Qlik Predict, or asks for help writing or fixing Qlik script syntax. Also trigger when the user provides a Qlik load script and asks for modifications, extensions, feature engineering, or data transformations. Even if the user just pastes Qlik script code and asks a question about it, use this skill."
license: Apache-2.0
metadata:
  author: nabeel-oz
  version: 1.5.0
  tags:
    - qlik
    - load-script
    - qvs
    - etl
    - data-preparation
    - qlik-predict
    - claude-code
    - cursor
    - coding-agent
---

# Qlik Load Script Generator

## Overview

You are writing **Qlik Sense load script** (.qvs files) for data preparation. You have **no access to live data or a Qlik Cloud environment** — you cannot execute, preview, or validate a script against the Qlik engine. Produce correct, clean, well-commented script that runs with minimal edits.

**Built for coding agents.** This skill is designed primarily for agentic coding tools with file-system access — Claude Code, Cursor, Codex, and similar — working inside a project that contains `.qvs` files:
- Read the existing script(s) from disk (via your file-read tool) before editing.
- Edit or create the `.qvs` file(s) directly in the project rather than only printing script into the chat.
- Preserve the surrounding project structure (other tabs, connections, file layout) exactly as found, except where the task asks you to change it.

If you are running in a chat-only environment with no file access (e.g. a plain web chat), fall back to outputting complete script blocks for the user to copy and paste — see [Workflow](#workflow).

**Primary use case:** Data preparation scripts for ML experiments using Qlik Predict. The user provides a starting script with base data. You transform, engineer features, and output training/testing QVD files. All guidance for this — target requirements, split patterns, leakage checks, script template — is in [references/ml-data-prep.md](references/ml-data-prep.md).

**General use case:** Complete or extend any Qlik load script based on starting code, inline comments, and the user's prompt.

## Critical Rules

1. **This is Qlik load script, not SQL.** The syntax resembles SQL but differs in important ways. When unsure, read [references/syntax-and-patterns.md](references/syntax-and-patterns.md) and consult: https://help.qlik.com/en-US/cloud-services/Subsystems/Hub/Content/Sense_Hub/LoadData/script-syntax-functions.htm
2. **Never invent functions or guess arguments.** If you are not confident a built-in function exists, check the Help docs, search Qlik Community, and use an approach you are confident about.
3. **No execution.** You cannot run or test the script against Qlik Cloud, regardless of environment. Flag anything you are uncertain about with a `// TODO: verify` comment so the user can check.
4. **Preserve the user's starting script.** Do not rewrite or restructure parts of the script the user did not ask you to change.
5. **Output complete, runnable script.** Do not output partial snippets with "..." elisions unless the user explicitly asks for a diff. When you have file access, write the complete section back to the `.qvs` file; when outputting to chat, assume the user is copy-pasting into the Qlik Cloud script editor.

**Before writing any Qlik script**, read [references/syntax-and-patterns.md](references/syntax-and-patterns.md) for the full syntax reference, common pitfalls, and code patterns. This is essential — Qlik syntax has many subtle differences from SQL that cause silent bugs if you rely on SQL intuition.

### Non-negotiable syntax rules

These five account for the majority of observed failures. They are all cases where SQL intuition produces script that either will not parse or is silently wrong. **Verify each one explicitly before returning any script** — do not rely on having "written it carefully".

1. **No `Count(*)`.** It is a syntax error. Use `Count(1)` to count all rows, `Count(FieldName)` to count non-null values of a field, `Count(DISTINCT FieldName)` for unique values, and `Sum(If(cond, 1, 0))` for a conditional count. `Count(1)` and `Count(Field)` differ by the number of nulls — choose deliberately, because the difference silently changes any ratio built on top.

2. **An alias cannot be used in the LOAD that creates it.** Per Qlik's docs, *"fields created through the `as` clause are out of scope and cannot be used inside the same load statement."* If field B is derived from field A, and A is created in this LOAD, they must be in **separate** LOADs — a preceding LOAD (stacked, executes bottom-up) or a `RESIDENT` step. Build calculation chains in explicit steps.

3. **Assume auto-concatenation unless explicitly ruled out.** Any LOAD whose field set matches an earlier table is silently appended to that table and **your table label is never created** — every later `RESIDENT`, `DROP TABLE`, `STORE`, or `JOIN (…)` on that name then fails at reload. The trigger is matching field names and count, not the table name or the data, so `LOAD * RESIDENT X` into a new name is always a candidate. Put `NoConcatenate` on every such LOAD (label first, then `NoConcatenate`, then `LOAD`). Before writing any table reference, name the LOAD that created it and confirm it survived under that name.

4. **No `HAVING`.** It does not exist in Qlik. To filter on an aggregate, aggregate in one LOAD and filter in a **second** pass — a preceding LOAD with `WHERE`, or a `RESIDENT` reload. A `WHERE` on a LOAD filters input rows *before* aggregation and cannot contain aggregation functions.

5. **`Round()`'s second argument is a step (interval), not a number of decimal places.** `Round(x, 3)` rounds to multiples of 3. For 2 decimals use `Round(x, 0.01)`; for 3, `Round(x, 0.001)` — for *N* decimals the step is `1/10^N`, always less than 1. Same for `Ceil()` and `Floor()`. If you have written a step ≥ 1, confirm you meant to bucket. For ML features, prefer not rounding at all — it discards signal.

**Then run the self-check** in [§14 of the reference](references/syntax-and-patterns.md#14-sql-habits-that-break-in-qlik), which also lists the wider set of SQL constructs that do not exist in Qlik (`ON` clauses, subqueries, CTEs, `CASE`, `UNION`, `COALESCE`, `LIMIT`, `OVER (PARTITION BY …)`, `%`, `+` for strings).

---

## Commenting Style

Write comments as an experienced Qlik developer would for colleagues who will maintain this script — not as a narration of your own work. The reader is a competent Qlik developer: they can read `LOAD`, `RESIDENT`, and `Sum()`. What they cannot recover from the code is **why**.

**Where comments belong**
- **Header block** (top of the script) — purpose, target/grain if ML, source data, owner.
- **Start of each `///$tab`** — one or two lines on what this section produces and why it exists.
- **Start of each logical block** (a table build, a join sequence, an aggregation) — the intent and any non-obvious logic or business rule behind it.
- **Individual lines** — only where there is a genuine point to make: a business rule or threshold and its rationale, a workaround with its reason, a non-obvious function argument, a unit or grain that isn't apparent, or a field description that saves the reader cross-referencing another part of the script. Deriving a field whose meaning isn't obvious from its name is a good reason; `// Load the customers table` is not.

**Register and length**
- Succinct and factual. One line where one line does. Full sentences are fine; paragraphs should be avoided.
- Explain **why**, not **what**. If a comment restates the code, delete it.
- No emoji, no decorative ASCII beyond the existing section banners, no enthusiasm, no hedging.
- Prefer the imperative or plain declarative: `// Exclude staff accounts — they distort the churn base rate.`

**Comments must not accumulate**
This is the failure mode to guard against hardest. When you revise a script:
- **Edit the existing comment to describe the new state.** Do not append a correction, a "fixed:" note, a "previously we…", or a second comment beside the first.
- **Delete comments describing code you removed.** A comment surviving its code is worse than no comment.
- **Never leave development history inline** — no `// changed 2026-08-11`, no `// was Count(*), now Count(1)`, no commented-out previous versions, no bug narration. The comment describes the code as it stands, in the present tense, as though written once.
- Ask of every comment you leave behind: *would a developer seeing this file for the first time, with no knowledge of its revision history, find this useful?* If not, it goes.

**Change log**
For complex or long-lived scripts, keep revision history in a dedicated `///$tab Change Log` placed early in the script (immediately after `Main`), not scattered through the code:

```qlik
///$tab Change Log
/**
 * 2026-08-11  NK  Added policy tenure and claims-frequency features.
 * 2026-07-02  NK  Switched split to time-based on RenewalDate (was random).
 * 2026-06-18  NK  Initial version.
 */
```

Add an entry here when making a substantive change to an existing script that already has this tab; create the tab when a script grows past a handful of tabs or when the user asks for one. Short scripts do not need it — do not add ceremony to a 30-line script.

**`// TODO: verify` notes** are the exception to all of the above: they are temporary, must state the reason, and the user removes them once checked.

---

## Workflow

When the user provides a starting script or prompt:

1. **Load the right reference(s).** Always read [references/syntax-and-patterns.md](references/syntax-and-patterns.md) before writing script. If the task is preparing data for a **Qlik Predict** experiment, also read [references/ml-data-prep.md](references/ml-data-prep.md) — it holds the target requirements, feature-engineering and leakage guidance, train/test split patterns, the explicit feature-matrix and Feature Definitions QVD requirements, script template, and final cleanup checklist.
2. **Understand the task.** For ML prep, state back the prediction target, grain, and key transformations. Ask clarifying questions only if the intent is genuinely ambiguous.
3. **Read the existing script(s) from disk** before editing, and note the lib connection paths and source formats already in use — match them.
4. **Track the tables.** As you write, keep a running inventory of every table: its name, its field set, whether it carries `NoConcatenate`, and where it is dropped. Consult it before every `RESIDENT`, `JOIN`, `STORE`, or `DROP` rather than assuming the name you wrote earlier exists. Two live tables with the same field set means one of them has been auto-concatenated away.
5. **Write the script.** With file access, edit or create the `.qvs` file(s) directly. Otherwise output complete, runnable sections in the chat. Comment per the [commenting style](#commenting-style) above.
6. **Re-read what you wrote and run the self-check** from [§14 of the syntax reference](references/syntax-and-patterns.md#14-sql-habits-that-break-in-qlik). Since you cannot execute the script, this pass is the only validation it gets — treat it as required, not optional. Check the script as written on disk, not your memory of writing it. In the same pass, re-read the comments: delete any that restate the code, describe code you removed, or narrate the revision you just made.
7. **Flag uncertainties** with `// TODO: verify — [reason]`.
8. **Explain non-obvious logic** after the script: complex `Window()` calls, tricky joins, domain-specific choices.
9. **Suggest improvements** — additional features, null/outlier handling, leakage risks.

---

## Reference

### Bundled references
- **[references/syntax-and-patterns.md](references/syntax-and-patterns.md)** — full Qlik load script syntax, common pitfalls, feature engineering code patterns, and the SQL-habits self-check (§14). **Read before writing any script.**
- **[references/ml-data-prep.md](references/ml-data-prep.md)** — Qlik Predict target requirements, feature/field retention guidance, date conventions, explicit feature matrix, Feature Definitions QVD, train/test split patterns, leakage and class-imbalance checks, script template, final cleanup checklist, worked example. **Read for any Qlik Predict data prep task.**

### Qlik documentation
- **Qlik Script Syntax & Functions:** https://help.qlik.com/en-US/cloud-services/Subsystems/Hub/Content/Sense_Hub/LoadData/script-syntax-functions.htm
- **Qlik Predict documentation:** https://help.qlik.com/en-US/cloud-services/Subsystems/Hub/Content/Sense_Hub/AutoML/home-automl.htm
- **Window function reference:** https://help.qlik.com/en-US/cloud-services/Subsystems/Hub/Content/Sense_Hub/LoadData/window-functions.htm
- **LOAD statement (authoritative clause list; alias scope rule):** https://help.qlik.com/en-US/cloud-services/Subsystems/Hub/Content/Sense_Hub/Scripting/ScriptRegularStatements/Load.htm
- **Count():** https://help.qlik.com/en-US/cloud-services/Subsystems/Hub/Content/Sense_Hub/Scripting/BasicAggregationFunctions/Count.htm
- **Round():** https://help.qlik.com/en-US/cloud-services/Subsystems/Hub/Content/Sense_Hub/Scripting/NumericFunctions/round.htm
- **NoConcatenate / automatic concatenation:** https://help.qlik.com/en-US/cloud-services/Subsystems/Hub/Content/Sense_Hub/Scripting/ScriptPrefixes/NoConcatenate.htm
