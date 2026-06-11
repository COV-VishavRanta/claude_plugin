---
name: runner
description: QA pipeline skill. Invoke this skill whenever the user sends any message or request — no specific command required. Handles optional recording, test planning, generation, execution, healing, and reporting. Stops after every phase so the QA can review before the next step.
applyTo: "**"
---

# 🚀 QA Runner (Review-Gated Mode)

Execute the QA pipeline below whenever the user sends **any message**. No specific command is required to start. **Each phase stops completely when its output is ready. The QA must explicitly trigger the next step.**

---

# 🧠 Core Principles

- **Stop after every phase** — each phase terminates completely when its output is ready; never auto-advance to the next step
- After every phase, display the output summary and the **exact phrase the QA must say** to continue
- Prefer tool invocation over manual commands
- Only load files required for the current step
- Cache and reuse context across steps — never re-read unchanged files
- ask user to confirm before proceeding to the next step, allowing them to review outputs and make manual adjustments if needed
- ask user to record flows only after the story context is loaded, so they can make informed decisions about what to record based on the specific scenarios in the story

---

# ⚙️ Real Tool Mappings

| Intent               | Tool                                                                 |
| -------------------- | -------------------------------------------------------------------- |
| Ask user a question  | `Ask Questions`                                                    |
| Check file exists    | `Glob`                                                               |
| Run Playwright tests | Use terminal tool (sync): `npx playwright test --project=chromium`   |
| Open HTML report     | Use terminal tool (async)                                            |
| Record flows         | Use terminal tool (async)                                            |
| Compile check        | Use terminal tool (sync): `npx tsc --noEmit --project tsconfig.json` |

---

# 🧠 Context Optimization

### ✅ Minimal File Access

- Do NOT scan entire repo
- Load only required files per step

### ✅ Cache and Reuse

- Config files → read once, reuse across all steps
- Test plan → generated once, passed directly to generator
- `config.json` values → read once in Step 1, never re-read

### ✅ Targeted Reads

- Generator → only test plan + DOM model (if available)
- Healer → only failing test file + error context
- Recordings → only if they exist

---

# ⚙️ Step 1: Environment Validation (Fast Gate)

Read `.claude/agents/config.json` once. Validate:

| Variable         | Required    | Action if Missing                                               |
| ---------------- | ----------- | --------------------------------------------------------------- |
| `TEST_BASE_URL`  | Yes         | STOP: "Set TEST_BASE_URL in .claude/agents/config.json"         |
| `adoProjectName` | Yes         | STOP: "Set adoProjectName in .claude/agents/config.json"        |
| `TEST_USERNAME`  | Conditional | Required only when reusable browser session auth is unavailable |
| `TEST_PASSWORD`  | Conditional | Required only when reusable browser session auth is unavailable |

> ⚠️ **DO NOT read `TEST_STORY_ID` from `config.json`** — the story ID is always resolved from the user's initial prompt or via interactive prompt in Step 1.5.

Also verify `TEST_BASE_URL` does NOT contain a path suffix like `/users/login`. If it does, warn:
"⚠️ TEST_BASE_URL should be the root domain (e.g., https://example.com), not a specific page."

For auth-capable projects, use `tests/auth.setup.ts` as the single auth bootstrap source and persist Playwright auth state to `storageState.json`. Login and signup feature tests should still start with `test.use({ storageState: undefined });` when intentionally validating unauthenticated behavior. In `microsoft` mode, `auth.setup.ts` should prefer reusing an existing Chrome/Microsoft session and only fall back to credential entry when reusable session auth is unavailable.

---

# 🔢 Step 1.1: Resolve Story ID and Work Item Type (ALWAYS INTERACTIVE)

**NEVER read `TEST_STORY_ID` from `config.json`.** Always resolve the story ID using this order:

1. **Initial prompt** — if the user provided a story ID in their message (e.g., `98915`), use it directly
2. **Interactive fallback** — if no ID was in the initial prompt, use `AskUserQuestion` to ask:
   ```
   header: story-id
   question: Enter the ADO Work Item / Story ID:
   ```

Store: `TEST_STORY_ID = <resolved value>`

If no ID can be resolved → STOP with message: "No story ID provided. Please provide a story ID to continue."

**Then ask for work item type** using `vscode/askQuestions`:

```
header: work-item-type
question: What type of work item is this?
options:
  - label: "User Story"
    description: "ADO User Story work item"
    recommended: true
  - label: "Feature"
    description: "ADO Feature work item"
```

Store:
- `WORK_ITEM_TYPE = "User Story"` (if User Story selected) or `"Feature"` (if Feature selected)
- `DOCS_BASE_PATH = docs/test_cases/user_stories` (if User Story) or `docs/test_cases/features` (if Feature)

---

# 🎥 Step 2: Recording

Skip for now — recording is offered after the story context is loaded (Step 3.5), so the QA can make an informed decision based on the specific scenarios in the story.

---

# 🔐 Step 2.1: ADO Access Check (Before Story Context)

Before reading story context from ADO, ensure the ADO integration is available.

Required outcome:

- The ADO MCP integration is reachable for work item reads
- If ADO is unavailable, the pipeline may still continue only when `{DOCS_BASE_PATH}/{TEST_STORY_ID}/user_story.md` already exists

| Condition                                     | Action                                                                              |
| --------------------------------------------- | ----------------------------------------------------------------------------------- |
| ADO available                                 | ✅ Continue to Story Context                                                        |
| ADO unavailable and story file already exists | ✅ Continue using the existing story file                                           |
| ADO unavailable and story file is missing     | STOP: `Could not load story context from ADO, and no local user story file exists.` |

---

# 🧠 Step 3: Story Context

Check if `{DOCS_BASE_PATH}/{TEST_STORY_ID}/user_story.md` exists.

| Condition    | Action                                                                                                                   |
| ------------ | ------------------------------------------------------------------------------------------------------------------------ |
| File exists  | ✅ Invoke `ado-context` skill — it will compare the revision number from ADO and only update if the revision has changed |
| File missing | Invoke `ado-context` skill to fetch from ADO and write the file                                                          |

Pass `WORK_ITEM_TYPE` and `DOCS_BASE_PATH` to the `ado-context` skill so it validates the work item type and saves to the correct path.
---

# 🎥 Step 3.1: Record or Skip? (ALWAYS ASK)

After story context is loaded, **always** ask the user before proceeding to test planning:

```
header: record_prompt
question: 🎥 Do you want to record user flows before generating the test plan?
options:
  - label: "Yes, record flows first"
    description: "Launch Playwright Codegen to capture real interactions with the app"
  - label: "No, skip recording"
    description: "Proceed directly to test planning using the story context"
    recommended: true
```

| Answer                  | Action                                                                               |
| ----------------------- | ------------------------------------------------------------------------------------ |
| Yes, record flows first | Invoke the `record` skill → wait for recording to complete → then continue to Step 4 |
| No, skip recording      | ✅ Continue directly to Step 4                                                       |

### 🛑 Stop After Recording (only if user chose to record)

Once recording finishes, display:

```
🛑 Recording complete. Stopped for QA review.
   • Recorded file: tests/recordings/{TEST_STORY_ID}/recorded-<timestamp>.spec.ts
   • Review the recorded flow before proceeding.

▶ To continue to test planning: say "generate test plan"
```

Then **stop completely**. Do not proceed to Step 4 until the QA explicitly says "generate test plan".

---

# 🧠 Step 4: Test Planning

Call subagent: `COV: Testing | 10 - QA Orchestrator`

Input:

- `{DOCS_BASE_PATH}/{TEST_STORY_ID}/user_story.md`
- `tests/recordings/` (only if recordings exist)

Output: `{DOCS_BASE_PATH}/{TEST_STORY_ID}/test-plans/test-plan.md`

### Step 4.1: Resolve Feature Name

From `{DOCS_BASE_PATH}/{TEST_STORY_ID}/test-plans/test-plan.md` first line matching `# Feature: <name>`:

- Slugify: lowercase, replace spaces with hyphens, strip special chars
- Example: `# Feature: Login` → `login`
- Fallback: `FEATURE_NAME` from `config.json` (`adoProjectName` slugified) → folder name under `tests/`

Store: `FEATURE_NAME = <resolved value>`

### 🛑 Stop After Test Planning

Once `{DOCS_BASE_PATH}/{TEST_STORY_ID}/test-plans/test-plan.md` is written, display:

```
🛑 Test plan generated. Stopped for QA review.
   • File: {DOCS_BASE_PATH}/{TEST_STORY_ID}/test-plans/test-plan.md
   • Open the file and review all scenarios before proceeding.

▶ To continue: say "generate test scripts"
```

Then **stop completely**. Do not proceed to Step 5 until the QA explicitly says "generate test scripts".

---

# 🤖 Step 5: Test Generation

Call subagent: `playwright-test-generator`

Input:

- `{DOCS_BASE_PATH}/{TEST_STORY_ID}/test-plans/test-plan.md`
- Recordings (if exist)

Output: `tests/<FEATURE_NAME>/*.spec.ts`

### 🛑 Stop After Test Script Generation

Once all test files are written, list every generated file, then display:

```
🛑 Test scripts generated. Stopped for QA review.
   • Test plan:     docs/test_cases/user_stories/{TEST_STORY_ID}/test-plans/test-plan.md
   • Generated files: tests/<FEATURE_NAME>/
     - (list each file here)
   • Review selectors, assertions, and test logic before running.

▶ To continue: say "run tests"
```

Then **stop completely**. Do not proceed to Step 5.1 or Step 6 until the QA explicitly says "run tests".

---

# ✅ Step 5.1: Pre-Execution Validation (NEW — prevents broken tests from running)

Before compiling or running tests, make auth setup explicit for any feature that depends on an authenticated browser session:

- If the feature under test is a login/authentication feature and tests intentionally start with `test.use({ storageState: undefined });`, skip `npm run auth:setup`
- If the feature is not a login/authentication feature and tests require authenticated state, run `npm run auth:setup`
- `npm run auth:setup` must execute `tests/auth.setup.ts` and create `storageState.json` when it is missing or stale
- Resolve auth setup in this order:
  - Prefer reusing an existing Chrome/Microsoft browser session and persist Playwright auth state to `storageState.json`
  - If reusable session auth is unavailable, use `TEST_USERNAME` and `TEST_PASSWORD` from `.claude/agents/config.json`
- If reusable session auth is unavailable and credentials are missing, STOP with: `For authenticated tests, either sign into the target account in local Chrome first or set TEST_USERNAME and TEST_PASSWORD in .claude/agents/config.json.`

Run: `npx tsc --noEmit --project tsconfig.json`

| Exit Code | Action                                                               |
| --------- | -------------------------------------------------------------------- |
| 0         | ✅ All tests compile — continue to Step 6                            |
| Non-zero  | Parse errors → feed to healer for immediate fix BEFORE running tests |

This catches:

- `.then()` on synchronous methods
- Missing imports
- Type errors
- Syntax errors

---

# 🚀 Step 6: Execute Tests

Scope the test run to the generated feature folder to avoid running unrelated tests (`example.spec.ts`, `seed.spec.ts`, or tests from other stories):

- **Login / auth feature** (`FEATURE_NAME = login`): `npx playwright test --project=chromium tests/login`
- **All other features**: `npx playwright test --project=chromium tests/<FEATURE_NAME>`

Wait for completion. Check exit code.

| Exit Code | Action                           |
| --------- | -------------------------------- |
| 0         | ✅ All tests pass → go to Step 8 |
| Non-zero  | Go to Step 7                     |

---

# 🔁 Step 7: Heal Failures

Trigger ONLY if Step 6 failed.

### 🛑 Stop Before Healing

Display all failures, then:

```
🛑 Tests failed. Stopped for QA review.
   • Failed tests:
     - (list each failing test name and its error)
   • Review failures before healing begins.

▶ To auto-heal: say "heal tests"
▶ To fix manually: edit the files yourself, then say "run tests"
```

Then **stop completely**. Do not invoke the healer until the QA explicitly says "heal tests".

When the QA says "heal tests", call subagent: `playwright-test-healer`

Input:

- ONLY the failing test files (from test-results/ error-context.md files)
- The specific error messages and page snapshots

Constraints:

- Max retries: 2
- Fix one test at a time
- Do NOT modify passing tests
- Do NOT re-run the entire suite — only re-run fixed tests individually first

### 🛑 Stop After Healing

Once the healer applies its fixes, list every changed file, then display:

```
🛑 Healer applied fixes. Stopped for QA review.
   • Modified files:
     - (list each file and a one-line summary of the change)
   • Review the fixes before re-running the suite.

▶ To re-run all tests: say "run tests"
```

Then **stop completely**. Do not re-run anything until the QA explicitly says "run tests".

---

# 📊 Step 8: Report

Delegate to the `report` skill with `FEATURE_NAME`.

Display summary and stop:

```
🛑 Report generated. Stopped for QA review.
   • Story:     {TEST_STORY_ID}
   • User story: {DOCS_BASE_PATH}/{TEST_STORY_ID}/user_story.md
   • Test plan:  {DOCS_BASE_PATH}/{TEST_STORY_ID}/test-plans/test-plan.md
   • Tests:      X passed, Y failed
   • Report:     playwright-report/{TEST_STORY_ID}-report/index.html

▶ To re-run failed tests only: say "re-run failed tests"
▶ To re-run full suite: say "run tests"
▶ Pipeline complete — no further action required.
```

Then **stop completely**.

---

# 🧹 Step 9: No Cleanup

The user story and test plan are **retained** after pipeline completion. Do NOT delete any files.

The following files persist for future reference:

- `{DOCS_BASE_PATH}/{TEST_STORY_ID}/user_story.md`
- `{DOCS_BASE_PATH}/{TEST_STORY_ID}/test-plans/test-plan.md`

---

# 📋 Pipeline Summary

| Step | Output                                                                 | Stops After?              | QA Trigger to Continue      |
| ---- | ---------------------------------------------------------------------- | ------------------------- | --------------------------- |
| 1    | config.json validated                                                  | No (auto-continues)       | —                           |
| 1.5  | Story ID resolved (prompt or argument)                                 | No (auto-continues)       | —                           |
| 2    | (pass-through)                                                         | No (auto-continues)       | —                           |
| 2.5  | ADO access validated or local story fallback confirmed                 | No (auto-continues)       | —                           |
| 3    | Story context file (revision-checked)                                  | No (auto-continues)       | —                           |
| 3.5  | Record or skip prompt                                                  | 🛑 Yes (if record chosen) | "generate test plan"        |
| 4    | `{DOCS_BASE_PATH}/{TEST_STORY_ID}/test-plans/test-plan.md` | 🛑 Yes                    | "generate test scripts"     |
| 5    | `tests/<feature>/*.spec.ts`                                            | 🛑 Yes                    | "run tests"                 |
| 5.1  | TypeScript validation                                                  | No (auto-continues)       | —                           |
| 6    | Test results                                                           | 🛑 Yes (on failure)       | "heal tests" or "run tests" |
| 7a   | Failure summary                                                        | 🛑 Yes                    | "heal tests"                |
| 7b   | Healed files                                                           | 🛑 Yes                    | "run tests"                 |
| 8    | HTML report                                                            | 🛑 Yes                    | Pipeline ends               |
| 9    | Files retained (no cleanup)                                            | No (auto-runs)            | —                           |

**Every 🛑 stop requires an explicit QA command to advance.**
