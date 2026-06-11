---
name: wi-change-detection
description: "Detect whether an agent's output file already exists and whether it needs to be regenerated based on ADO work item changes. Use when: any agent has obtained a Work Item ID and needs to decide between CREATE, UPDATE, or SKIP before performing expensive reads. Returns a per-file mode map and short-circuits the calling agent when output is already current."
argument-hint: "WI data (pre-fetched), output file path(s) to check, and relevant WI fields (Title, Description, AC, Attachments)"
---

# WI Change Detection

## Purpose

Determine the correct write mode (CREATE / UPDATE / SKIP) for each of a calling agent's output files, using the ADO work item data the agent has already fetched. The skill performs a fast header-only read of existing output files and compares content fingerprints, so calling agents can short-circuit expensive architecture doc reads and codebase scans when nothing has meaningfully changed.

## When to Use

- An agent has a Work Item ID and needs to decide whether to generate output files from scratch, update them, or skip entirely
- Before reading architecture docs, feature context, or scanning the codebase
- Any agent that produces output files tied to an ADO work item

## Required Tools

| Tool Namespace | Purpose                                                                       |
| -------------- | ----------------------------------------------------------------------------- |
| `read`         | Read the first 10 lines of existing output files (header-only)                |
| `edit`         | Patch `ado-rev` in the file header on non-impacting SKIP (minimal write only) |

> `ado/*` tools are **not** needed — the calling agent must supply pre-fetched WI data. Zero extra API calls are made by this skill.

---

## Input

The calling agent must provide:

| Parameter             | Required | Description                                                                                                                                         |
| --------------------- | -------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| **WI data**           | Yes      | Pre-fetched work item object from `ado/get_work_item`. Must include: Title, Rev, Description, Acceptance Criteria (if applicable), Attachments list |
| **Output file path(s)** | Yes    | One or more file paths to check. Checked in parallel when multiple paths are provided                                                               |
| **Relevant fields**   | Yes      | List of WI fields that affect this agent's output. Features: `[Title, Description, Attachments]`. User Stories: `[Title, Description, AcceptanceCriteria, Attachments]` |

---

## Output

A **per-file mode map** returned to the calling agent:

```
{
  "docs/feature-plans/123-my-feature/workflow.md": "SKIP",
  "docs/.../frontend-implementation-plan.md": "CREATE",
  "docs/.../backend-implementation-plan.md": "UPDATE"
}
```

Accompanied by a **user-facing message** explaining the result for each file.

---

## Content Fingerprint Format

Calling agents must store these fields in every output file's metadata header (immediately after `ado-rev`). The skill reads and compares them during UPDATE detection.

| Header Field                       | Value Format                                                                                                                                                       | WI Field              | Used By              |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ | --------------------- | -------------------- |
| `content-fingerprint-title`        | First 50 chars of Title (trimmed)                                                                                                                                  | Title                 | All agents           |
| `content-fingerprint-description`  | First 100 chars + `...` + last 50 chars of Description (trimmed). If Description is ≤ 160 chars, store full content without ellipsis                              | Description           | All agents           |
| `content-fingerprint-ac`           | First 100 chars + `...` + last 50 chars of Acceptance Criteria (trimmed). If AC is ≤ 160 chars, store full content without ellipsis                               | AcceptanceCriteria    | User Story agents only |
| `content-fingerprint-attachments`  | `{count}:{sorted comma-separated attachment file names}`. Example: `2:design.png,wireframe.pdf`. Use `0:none` when no attachments                                  | Attachments           | All agents           |

> **Why bookend fingerprints?** Long descriptions that change only near the end would be missed by a prefix-only approach. The first 100 + last 50 chars catches changes at both extremes while staying lightweight and not requiring full content storage.

**Example header block (feature workflow):**

```
> ado-rev: 14
> content-fingerprint-title: Manage Webhook Notification Credential
> content-fingerprint-description: Users need a way to configure webhook endpoints for external system...credentials should be stored encrypted and rotatable on demand.
> content-fingerprint-attachments: 1:webhook-flow-design.png
```

**Example header block (implementation plan):**

```
> ado-rev: 7
> content-fingerprint-title: As a user I can view my webhook settings
> content-fingerprint-description: The settings page should display all configured webhooks with their...Each webhook row should show the URL, status, and last-triggered date.
> content-fingerprint-ac: Given I am on the Webhook Settings page, When the page loads, Then all...And each row displays a masked credential with a "Rotate" action button.
> content-fingerprint-attachments: 0:none
```

---

## Procedure

### Step 1 — Parallel File Existence Check

> **⚠️ CRITICAL: Do NOT use `file_search` with glob patterns for existence checks.** Glob-based searches (`file_search` with wildcards like `98829-*/workflow.md`) can silently return "No files found" even when the file exists, causing a false CREATE assignment that wastes expensive API calls. Use `read_file` (lines 1–10) on the **exact resolved path** instead — it reliably returns content if the file exists or an error if it does not.

Check all provided output file paths **in parallel** using `read_file` (lines 1–10) on each exact path. For each path:

- If `read_file` **errors** (file not found) → assign mode **CREATE** immediately for that path. No further checks needed for it.
- If `read_file` **succeeds** (returns content) → the file exists. The header content is now already available — proceed directly to Step 2 for that path (no additional read needed).

> **Optimization:** Run all existence checks simultaneously. Paths assigned CREATE skip Steps 2–3 entirely. Successful reads double as the header read for Step 2 — zero additional file I/O.

---

### Step 2 — Revision Match Check (Fast Path)

For each existing file, read **only the first 10 lines** (the metadata header). Extract the `ado-rev` value.

- If `ado-rev` **matches** the current WI `Rev` → assign mode **SKIP (exact rev match)** for that path. No content comparison needed.
- If `ado-rev` **does not match** → proceed to Step 3 for that path.

> **Optimization:** Exact rev match short-circuits before any content comparison. Only files with a rev mismatch proceed to Step 3.

---

### Step 3 — Content Fingerprint Comparison

For each file with a rev mismatch, extract all `content-fingerprint-*` fields from the already-read header.

> **Backward compatibility:** If any `content-fingerprint-*` field is **absent** in the existing file (generated before this skill was in use), treat that field as **changed** and assign mode **UPDATE**. The subsequent regeneration will write all fingerprint fields, making future runs use the full comparison. Do not attempt to infer missing fingerprints.

For each `content-fingerprint-*` field present, build the equivalent fingerprint from the current WI data (using the format table above) and compare character-by-character:

| Comparison Result          | Action                                                                                                                                                                                                                                         |
| -------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **All fingerprints match** | Assign mode **SKIP (non-impacting change)**. Patch only the `ado-rev` line in the file header using `multi_replace_string_in_file` (single-line replacement — do not touch any other content). Print: `✅ Non-impacting changes detected (e.g. State/Assignment update). ado-rev updated to {N}. No content regeneration needed.` |
| **Any fingerprint differs** | Assign mode **UPDATE**. Do not modify the file — generation happens in the calling agent's generation step.                                                                                                                                     |

> **Optimization:** Fingerprint comparison is pure string matching against already-read header data — no additional file reads or API calls required.

---

### Step 4 — Return Mode Map

Collect all per-file mode assignments and return them to the calling agent as a mode map. Include a user-facing message summarising what was found.

**Caller responsibilities after receiving the map:**

- **All files SKIP** → halt immediately, print the ✅ messages, do not proceed further
- **Any file is CREATE or UPDATE** → proceed with the pipeline, applying each file's pre-determined mode at generation time
- **SKIP files at generation time** → skip generation entirely for those specific files; do not overwrite them

---

## Optimization Summary

| Strategy                                    | Benefit                                                                               |
| ------------------------------------------- | ------------------------------------------------------------------------------------- |
| Zero extra ADO API calls                    | WI data is always pre-fetched by the calling agent before invoking this skill         |
| Parallel file existence checks              | Multiple output files checked simultaneously                                          |
| Header-only reads (first 10 lines)          | Avoids loading full document content for revision and fingerprint extraction          |
| Short-circuit on exact rev match            | Skips fingerprint comparison entirely when rev is identical                           |
| Bookend fingerprints (first 100 + last 50 chars) | Catches changes at both ends of long content without storing full text           |
| Minimal write on non-impacting SKIP         | Only the `ado-rev` line is patched — prevents redundant re-checking on next run      |
| Mode map carries forward                    | Pre-computed modes travel through the calling agent's pipeline; no re-detection later |
