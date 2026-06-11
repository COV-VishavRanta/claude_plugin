---
name: record
description: "Record user interaction flows using Playwright Codegen against the TEST_BASE_URL defined in .claude/agents/config.json. Use when: recording a new user flow, capturing browser interactions, generating test recordings, running codegen, launching the recorder, capturing a user journey before running the QA pipeline."
argument-hint: "Optional: describe what flow to record (e.g. 'login flow', 'checkout journey')"
---

# 🎥 Record User Flows (Playwright Codegen)

Launches `playwright codegen` pointed at `TEST_BASE_URL` from `.claude/agents/config.json` and saves the recorded spec to `tests/recordings/`.

---

## ⚙️ Real Tool Mappings

| Intent                                  | Tool                         |
| --------------------------------------- | ---------------------------- |
| Check `config.json` exists              | `Glob`                       |
| Read `TEST_BASE_URL` from `config.json` | As needed                    |
| Launch recorder                         | Run on terminal tool (async) |
| Confirm user finished                   | `AskUserQuestion`            |
| Report output file                      | Text message to user         |

---

## Step 1: Verify `config.json` and `TEST_BASE_URL`

Use `Glob` to confirm `.claude/agents/config.json` exists at the project root.

Then use `read_file` on `.claude/agents/config.json` to extract `TEST_BASE_URL`.

| Condition                        | Action                                                                           |
| -------------------------------- | -------------------------------------------------------------------------------- |
| `config.json` missing            | STOP — instruct user to create `.claude/agents/config.json` with `TEST_BASE_URL` |
| `TEST_BASE_URL` not set or empty | STOP — instruct user to add `TEST_BASE_URL` to `.claude/agents/config.json`      |
| `TEST_BASE_URL` present          | ✅ Continue — display the URL to the user so they can confirm it                 |

---

## Step 2: Launch the Recorder

Use Run on terminal tool (async, no timeout) with:

```
npm run record
```

This runs `scripts/record.ts` which executes:

```
npx playwright codegen <TEST_BASE_URL> --output tests/recordings/{TEST_STORY_ID}/recorded-<timestamp>.spec.ts
```

Inform the user:

> 🎥 Playwright Codegen is open at **`<TEST_BASE_URL>`**. Interact with the app in the browser to record your flow. When done, **close the browser** to save the recording.

---

## Step 3: Confirm Recording Complete

Use `AskUserQuestion`:

```
header: recording_done
question: Have you finished recording and closed the browser?
options:
  - label: "Yes, recording is done"
    recommended: true
  - label: "Cancel / something went wrong"
```

| Answer | Action                                             |
| ------ | -------------------------------------------------- |
| Yes    | ✅ Continue to Step 4                              |
| Cancel | STOP — advise user to re-run `./record` when ready |

---

## Step 4: Confirm Output File

Use `Glob` with pattern `tests/recordings/{TEST_STORY_ID}/recorded-*.spec.ts` to find the most recently created file.

Report to the user:

> ✅ Recording saved to: `tests/recordings/{TEST_STORY_ID}/recorded-<timestamp>.spec.ts`
>
> You can now run `./runner` to kick off the full QA pipeline using this recording.

If no file is found → warn the user the recording may not have saved (browser may have been closed before any interactions were made).

---

## Notes

- `TEST_BASE_URL` is loaded from `.claude/agents/config.json` by `scripts/record.ts`
- Output files are timestamped and saved to `tests/recordings/`
- The runner pipeline (`./runner`) automatically picks up recordings from `tests/recordings/` in Step 4 (Orchestrator)
- Multiple recordings can exist — the orchestrator merges all of them
