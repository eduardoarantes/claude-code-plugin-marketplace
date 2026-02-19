---
model: opus
description: Parallel user story validation — discovers YAML stories, fans out bowser-qa-agents, aggregates results
argument-hint: [stories-dir] [headed] [filename-filter] [vision]
---

# Purpose

Discover user stories from YAML files, run optional setup (login/auth) agents first, then fan out parallel `bowser-qa-agent` instances to validate each story and aggregate pass/fail results with screenshots.

## Variables

STORIES_DIR: first argument that looks like a path (contains `/` or starts with `.`); defaults to `ai_review/user_stories`
STORIES_GLOB: "{STORIES_DIR}/*.yaml"
HEADED: "true" or "headed" keyword in arguments (default: "false")
VISION: "vision" keyword in arguments — screenshots returned as image responses in context (higher token cost). Default: false.
FILENAME_FILTER: remaining non-keyword, non-path arguments
AGENT_TIMEOUT: 300000
SCREENSHOTS_BASE: "screenshots/bowser-qa"
RUN_DIR: "{SCREENSHOTS_BASE}/{YYYYMMDD_HHMMSS}_{short-uuid}" (generated once at start of run)

## Argument Parsing

Parse `$ARGUMENTS` as follows (order does not matter):
- If an argument contains `/` or starts with `.` → it is `STORIES_DIR`
- If an argument is `"true"` or `"headed"` → `HEADED=true`
- If an argument is `"vision"` → `VISION=true`
- Any remaining arguments → `FILENAME_FILTER`

Examples:
- `/ui-review` → STORIES_DIR=`ai_review/user_stories`
- `/ui-review tests/stories` → STORIES_DIR=`tests/stories`
- `/ui-review headed` → STORIES_DIR=`ai_review/user_stories`, HEADED=true
- `/ui-review tests/stories athlete-login headed` → STORIES_DIR=`tests/stories`, FILENAME_FILTER=`athlete-login`, HEADED=true

## YAML File Format

Each YAML file may contain an optional `setup` array and a required `stories` array:

```yaml
# Optional: one entry per unique auth identity
setup:
  - auth_file: "auth/cameron.json"   # path where state will be saved
    url: "http://localhost:3000/login"
    workflow: |
      Navigate to the login page
      Fill email, fill password, click login
      Wait for redirect to dashboard

# Required
stories:
  - name: "Homepage loads"
    url: "http://localhost:3000"
    # no auth_file → runs without authentication
    workflow: |
      Navigate to the homepage
      Verify it loads successfully

  - name: "Dashboard loads"
    url: "http://localhost:3000/dashboard"
    auth_file: "auth/cameron.json"   # load this state before navigating
    workflow: |
      Verify the dashboard page loads
      Verify key widgets are visible
```

Stories without `auth_file` run as unauthenticated. Stories with `auth_file` load the saved state before navigating, skipping manual login.

## Codebase Structure

```
{STORIES_DIR}/          # YAML user story files (configurable, default: ai_review/user_stories)
auth/
└── *.json              # ephemeral auth state files (gitignored)
screenshots/
└── bowser-qa/
    └── 20260210_143022_a1b2c3/
        ├── {file-stem}/
        │   ├── {slugified-story-name}/
        │   └── ...
        └── report.md
```

## Instructions

- If FILENAME_FILTER is provided and non-empty, only run stories from files whose name contains that substring
- If a YAML file fails to parse, log a warning and skip it — do not abort the entire run
- If no stories are found after discovery, report that and stop
- Be resilient: if a teammate times out or crashes, mark that story as FAIL and include whatever output was available
- The `subagent_type` for each Task call must be `bowser-qa-agent`

## Workflow

### Phase 1: Discover

1. Use the Glob tool to find all files matching `STORIES_GLOB`
2. If `FILENAME_FILTER` is provided and non-empty, filter the file list to only include files whose name contains that substring
3. Read each YAML file and parse both the `setup` array (optional) and `stories` array
4. If a file fails to parse, log a warning and skip it
5. Build a flat list of all stories across all files, tracking which source file each story came from
6. Collect all unique `auth_file` values referenced by stories across all files (deduplicated)
7. If no stories are found, report that and stop
8. Generate `RUN_DIR` using Bash:
   ```bash
   RUN_DIR="screenshots/bowser-qa/$(date +%Y%m%d_%H%M%S)_$(uuidgen | tr '[:upper:]' '[:lower:]' | head -c 6)"
   ```
9. For each story, build its `SCREENSHOT_PATH`:
   - `{RUN_DIR}/{file-stem}/{slugified-story-name}/`
   - Example: `screenshots/bowser-qa/20260210_143022_a1b2c3/athlete-login/activities-page-loads/`
10. Create the `auth/` directory if any setup entries exist:
    ```bash
    mkdir -p auth
    ```

### Phase 1.5: Setup (Auth State)

Run this phase only if any stories reference an `auth_file`.

11. Collect all `setup` entries across all parsed YAML files. Deduplicate by `auth_file` path.
12. For each unique setup entry, spawn one `bowser-qa-agent` via the Task tool using this prompt:

```
You are performing a one-time login to save browser auth state for reuse by parallel test agents.

**Login URL:** {setup.url}
**Auth file to save:** {setup.auth_file}
**Headed:** {HEADED}

**Workflow:**
{setup.workflow}

After completing the workflow, save the browser state:
  playwright-cli state-save {setup.auth_file}

Instructions:
- Follow the workflow steps sequentially
- Do NOT take screenshots
- After saving the state, confirm the file was written
- Use this exact format for your final line:
  SETUP: {DONE|FAIL} | {setup.auth_file}
```

13. **Wait for ALL setup agents to complete before proceeding.** If any setup agent fails (reports `SETUP: FAIL`), mark all stories that depend on that `auth_file` as FAIL and exclude them from the spawn phase.

### Phase 2: Spawn

14. For each remaining story (not excluded by setup failure), spawn a `bowser-qa-agent` teammate via the Task tool. **Launch ALL story agents in a single message so they run in parallel.**

For stories **without** `auth_file`, use this prompt:

```
Execute this user story and report results:

**Story:** {story.name}
**URL:** {story.url}
**Headed:** {HEADED}
**Vision:** {VISION}

**Workflow:**
{story.workflow}

Instructions:
- Follow each step in the workflow sequentially
- Take a screenshot after each significant step
- Save ALL screenshots to: {SCREENSHOT_PATH}
- Report each step as PASS or FAIL with a brief explanation
- At the end, provide a summary: total steps, passed, failed
- Use this exact format for your final summary line:
  RESULT: {PASS|FAIL} | Steps: {passed}/{total}
```

For stories **with** `auth_file`, use this prompt:

```
Execute this user story and report results. Auth state is pre-saved — do NOT log in manually.

**Story:** {story.name}
**URL:** {story.url}
**Headed:** {HEADED}
**Vision:** {VISION}

**Pre-step (do this first, before anything else):**
Load saved auth state: playwright-cli state-load {story.auth_file}
Then navigate to: {story.url}

**Workflow:**
{story.workflow}

Instructions:
- Load auth state and navigate FIRST (pre-step above), then follow the workflow
- Do NOT perform a manual login — auth state is already saved
- Take a screenshot after each significant step
- Save ALL screenshots to: {SCREENSHOT_PATH}
- Report each step as PASS or FAIL with a brief explanation
- At the end, provide a summary: total steps, passed, failed
- Use this exact format for your final summary line:
  RESULT: {PASS|FAIL} | Steps: {passed}/{total}
```

### Phase 3: Collect

15. Wait for all story agent messages to arrive
16. Parse each report to extract:
    - Overall result: PASS or FAIL (look for `RESULT:` line)
    - Steps completed vs total (from `Steps: X/Y`)
    - The full agent report text
17. Include any stories that were pre-failed due to setup failure

### Phase 4: Cleanup and Report

18. Delete ephemeral auth state files:
    ```bash
    rm -f auth/*.json
    ```
19. Compose the full report markdown (see `Report` section below for the format).
20. Write the report to `{RUN_DIR}/report.md` using the Write tool.
21. Print the report to the terminal so the user can see the results inline.

## Report

Compose this markdown, write it to `{RUN_DIR}/report.md`, then also print it to the terminal:

```
# UI Review Summary

**Run:** {current date and time}
**Stories dir:** {STORIES_DIR}
**Stories:** {total} total | {passed} passed | {failed} failed
**Status:** ✅ ALL PASSED | ❌ PARTIAL FAILURE | ❌ ALL FAILED

## Results

| #   | Story        | Source File | Status | Steps            |
| --- | ------------ | ----------- | ------ | ---------------- |
| 1   | {story name} | {filename}  | ✅ PASS | {passed}/{total} |
| 2   | {story name} | {filename}  | ❌ FAIL | {passed}/{total} |

## Failures

(Only include this section if there are failures)

### Story: {failed story name}
**Source:** {filename}
**Agent Report:**
{full agent report for this story}

---

(Repeat for each failed story)

## Screenshots
All screenshots saved to: `{RUN_DIR}/`
```

Use ✅ ALL PASSED for status only when every story passed. Use ❌ PARTIAL FAILURE when some passed and some failed. Use ❌ ALL FAILED when none passed.
