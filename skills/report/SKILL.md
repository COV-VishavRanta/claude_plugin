---
name: report
description: "Open or link the Playwright HTML test report after a test run. Use when: tests have finished running, viewing test results, opening the playwright report, checking pass/fail summary, sharing the report path, after ./runner completes."
argument-hint: "Optional: feature name to open a specific report (e.g. 'login')"
---

# 📊 Report — Open or Link Playwright HTML Report

Detects available Playwright HTML reports, asks whether to launch the report server or just show the path.

---

## ⚙️ Real Tool Mappings

| Intent                  | Tool                                                                              |
| ----------------------- | --------------------------------------------------------------------------------- |
| Find available reports  | `file_search`                                                                     |
| Read summary JSON       | `read_file` on `playwright-report/{TEST_STORY_ID}-report/report.json` (if exists) |
| Resolve `TEST_STORY_ID` | read from `.claude/agents/config.json` → `TEST_STORY_ID`                          |
| Ask user open or skip   | `AskUserQuestion`                                                                 |
| Launch report server    | Run on terminal tool (async, no timeout)                                          |

---

## Step 1: Detect Available Reports

Use `file_search` with pattern `playwright-report/*-report/index.html` to list all available report folders.

Derive the list of story IDs from the matched paths (e.g. `playwright-report/98915-report/index.html` → `98915`).

| Condition          | Action                                                            |
| ------------------ | ----------------------------------------------------------------- |
| No reports found   | STOP — inform user no reports exist yet; run `./runner` first     |
| Exactly one report | Use it automatically — no need to ask                             |
| Multiple reports   | Use `AskUserQuestion` to let user pick which story report to open |

---

## Step 2: Resolve Story ID

If argument was provided (e.g. `./report 98915`) → use that directly.

Otherwise, resolve in this order (first match wins):

1. **`config.json` → `TEST_STORY_ID`** — read from `.claude/agents/config.json`.
2. **Report folder detected in Step 1** — strip `-report` suffix from folder name to get the story ID.
3. If still ambiguous → ask via `AskUserQuestion`.

Store: `STORY_ID = <resolved value>`
Report folder: `playwright-report/{STORY_ID}-report/`

---

## Step 3: Show Pass/Fail Summary

Use `file_search` to check if `playwright-report/{STORY_ID}-report/report.json` exists.

### If `report.json` exists:

Use `read_file` on it and extract:

- `stats.expected` → passed
- `stats.unexpected` → failed
- `stats.skipped` → skipped
- `stats.duration` → total duration (ms → seconds)

Print the summary **before** asking the user whether to open the report:

```
📋 Test Summary — Story #{STORY_ID}
   ✅ Passed:  <n>
   ❌ Failed:  <n>
   ⏭️ Skipped: <n>
   ⏱️ Duration: <n>s
```

If there are failures, also list each failing test title from `suites[*].specs[*]` where `ok === false`.

### If `report.json` does not exist:

Skip the summary silently — proceed to Step 4.

---

## Step 4: Ask — Open or Skip

Use `AskUserQuestion`:

```
header: open_report
question: 📊 Open test report?
options:
  - label: "Open Report"
    description: "Launch the Playwright HTML report in the browser"
    recommended: true
  - label: "Skip"
    description: "Just show the report path"
```

---

## Step 5: Act on Answer

### If "Open Report":

Use Run on terminal tool (async, **no timeout**) with:

```
npx playwright show-report playwright-report/{STORY_ID}-report
```

Inform the user:

> 🌐 Report is live at **http://localhost:9323** — press `Ctrl+C` in the terminal when done viewing.

### If "Skip":

Print the report path as a workspace-relative file link:

> 📄 Report available at: [playwright-report/{STORY_ID}-report/index.html](playwright-report/{STORY_ID}-report/index.html)

---

## Notes

- Report folders are named `playwright-report/{TEST_STORY_ID}-report/` — set by `playwright.config.ts` using `TEST_STORY_ID` from `config.json`
- Default folder is `playwright-report/default-report/` when `TEST_STORY_ID` is unset
- `npx playwright show-report` serves on port `9323` by default — change with `--port <n>` if that port is busy
- Reports persist between runs; older runs are overwritten when `TEST_STORY_ID` is the same
