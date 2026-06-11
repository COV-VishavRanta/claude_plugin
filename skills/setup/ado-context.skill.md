---
name: ado-context
description: "Fetch Azure DevOps user story context by ID. Fast path: accepts it as argument. Falls back to interactive selection."
argument-hint: "Story ID (e.g., 98915)"
---

# 🎯 ADO Context Provider — Optimized for Speed

## Priority: Direct Fetch by ID (Fast Path)

### Step 1: Resolve Story ID

Resolution order (first match wins):

1. **Argument** — if the user provided a story ID in the initial prompt (e.g., `98915`), use it directly
2. **Interactive fallback** — if no ID was provided, use the `AskUserQuestion` tool to ask the user:
   ```
   header: story-id
   question: Enter the ADO Work Item / Story ID:
   ```

Store: `storyId = <resolved value>`

If no ID can be resolved → STOP with message: "No story ID provided. Please provide a story ID to fetch context."

---

### Step 2: Resolve Project

Read `adoProjectName` from `.claude/agents/config.json`.

Store: `selectedProject = <adoProjectName value>`

---

### Step 3: Fetch Work Item (Single API Call)

Call `ado.getWorkItem` with:

```json
{
  "id": "{{storyId}}",
  "project": "{{selectedProject}}"
}
```

If the call fails or the work item is not found → STOP with message: "Work item #{{storyId}} was not found on Azure DevOps. Verify the ID and ensure Azure DevOps is accessible."

---

### Step 4: Extract & Normalize Context

From the work item response, extract:

- `id` — work item ID
- `title` — story title
- `state` — current state (New, Active, Closed, etc.)
- `description` — full description
- `acceptanceCriteria` — all acceptance criteria
- `priority` — priority level
- `assignedTo` — assignee (if available)
- `Story revision:` — for change tracking'

---

### Step 5: Save Story

Write to: `docs/test_cases/user_stories/{TEST_STORY_ID}/user_story.md`

**Revision Check (MANDATORY before writing):**

1. Check if `docs/test_cases/user_stories/{TEST_STORY_ID}/user_story.md` already exists
2. If it exists, read the line containing `*Story revision:` and extract the stored revision number
3. Compare with the `revision` value returned from the ADO API response (`fields['System.Rev']`)
4. **If revisions match** → Skip writing; log: `⏭️ Story #{{id}} is up to date (revision {{revision}}) — no changes needed`
5. **If revisions differ or file does not exist** → Write the file with updated content

Format:

```markdown
# User Story #{{id}} — {{title}}

**Project:** {{selectedProject}}
**State:** {{state}}
**Story Points:** {{storyPoints}}
**Priority:** {{priority}}

## Description

{{description}}

## Acceptance Criteria

{{acceptanceCriteria}}

## Impact Areas

{{impactAreas}}

---

_Fetched: {{timestamp}}_
_Story revision: {{revision}}_
```

---

### Step 6: Validate Against Recordings (Optional)

IF any file matching `tests/recordings/{TEST_STORY_ID}/recorded-*.spec.ts` exists:

- Compare acceptance criteria with the recorded flow actions
- If major mismatch detected, warn user and suggest reviewing the story or updating recordings

---

### Step 7: Output Summary

Display:

```
✅ ADO Context loaded
   • Story:    #{{id}} — {{title}}
   • Project:  {{selectedProject}}
   • State:    {{state}}
   • Revision: {{revision}}
   • Criteria: {{count}} acceptance criteria found
   • File:     docs/test_cases/user_stories/{TEST_STORY_ID}/user_story.md
```

---

## Performance Rules

- **NEVER** enumerate all projects unless absolutely necessary
- **NEVER** enumerate all stories unless absolutely necessary
- **Single API call** to fetch the work item by ID
- Total expected execution: < 5 seconds (network dependent)
- `adoProjectName` is always read from `config.json` — never ask the user for the project
- Story ID is always resolved from the initial prompt or interactively — never from `config.json`
