---
model: opus
description: Per-file user story validation — discovers YAML files, fans out one bowser-qa-agent per file, stories run sequentially within each agent
argument-hint: [main-site-url] [stories-dir] [headed] [filename-filter] [vision]
---

# Purpose

Discover user stories from YAML files, run optional setup (login/auth) agents first, then fan out one `bowser-qa-agent` per YAML file. Each agent runs the stories from its file sequentially and reports results.

## Variables

STORIES_DIR: first argument that looks like a path (contains `/` or starts with `.`); defaults to `ai_review/user_stories`
STORIES_GLOB: "{STORIES_DIR}/*.yaml"
HEADED: "true" or "headed" keyword in arguments (default: "false")
VISION: "vision" keyword in arguments — screenshots returned as image responses in context (higher token cost). Default: false.
FILENAME_FILTER: remaining non-keyword, non-path arguments
AGENT_TIMEOUT: 300000
SCREENSHOTS_BASE: "bowser-qa-test-results"
RUN_DIR: "{SCREENSHOTS_BASE}/{YYYYMMDD_HHMMSS}_{short-uuid}" (generated once at start of run)
MAIN_SITE: base URL for the application under test (e.g., `http://localhost:3000`). Resolved from `{RUN_DIR}/main_site` if it exists, otherwise prompted from the user and saved there.

## Argument Parsing

Parse `$ARGUMENTS` as follows (order does not matter):
- If an argument starts with `http://` or `https://` → it is `MAIN_SITE`
- If an argument contains `/` or starts with `.` (but is not a URL) → it is `STORIES_DIR`
- If an argument is `"true"` or `"headed"` → `HEADED=true`
- If an argument is `"vision"` → `VISION=true`
- Any remaining arguments → `FILENAME_FILTER`

Examples:
- `/ui-review` → STORIES_DIR=`ai_review/user_stories`
- `/ui-review http://localhost:3000` → MAIN_SITE=`http://localhost:3000`
- `/ui-review tests/stories` → STORIES_DIR=`tests/stories`
- `/ui-review headed` → STORIES_DIR=`ai_review/user_stories`, HEADED=true
- `/ui-review tests/stories http://localhost:3000 athlete-login headed` → STORIES_DIR=`tests/stories`, MAIN_SITE=`http://localhost:3000`, FILENAME_FILTER=`athlete-login`, HEADED=true

## YAML File Format

Each YAML file may contain an optional `setup` array and a required `stories` array:

```yaml
# Optional: one entry per unique auth identity
setup:
  - auth_file: "cameron.json"        # filename — saved to {RUN_DIR}/auth/
    url: "[MAIN_SITE]/login"
    workflow: |
      Navigate to the login page
      Fill email, fill password, click login
      Wait for redirect to dashboard

# Required
stories:
  - name: "Homepage loads"
    url: "[MAIN_SITE]"
    # no auth_file → runs without authentication
    workflow: |
      Navigate to the homepage
      Verify it loads successfully

  - name: "Dashboard loads"
    url: "[MAIN_SITE]/dashboard"
    auth_file: "cameron.json"        # must match a setup entry
    workflow: |
      Verify the dashboard page loads
      Verify key widgets are visible
```

Stories without `auth_file` run as unauthenticated. Stories with `auth_file` load the saved state from `{RUN_DIR}/auth/{auth_file}` before navigating, skipping manual login.

## Codebase Structure

```
{STORIES_DIR}/          # YAML user story files (configurable, default: ai_review/user_stories)
bowser-qa-test-results/
└── 20260210_143022_a1b2c3/
    ├── auth/
    │   └── *.json          # ephemeral auth state files (per-run, auto-cleaned)
    ├── {file-stem}/
    │   ├── {slugified-story-name}/
    │   └── ...
    └── report.md
```

## Instructions

- **NEVER attempt to fix, start, restart, or recover the application under test.** You are a test orchestrator, not a developer. If the server is down, a database is empty, or any infrastructure is broken — report the failure and stop. Do not run servers, seed databases, install dependencies, or modify any application code.
- If FILENAME_FILTER is provided and non-empty, only run stories from files whose name contains that substring
- If a YAML file fails to parse, log a warning and skip it — do not abort the entire run
- If no stories are found after discovery, report that and stop
- Be resilient: if a teammate times out or crashes, mark that story as FAIL and include whatever output was available
- The `subagent_type` for each Task call must be `bowser-qa-agent`
- **CRITICAL: Spawn exactly ONE agent per YAML file, NOT one agent per story.** Each agent receives ALL stories from its YAML file and executes them sequentially. If you have 3 YAML files, you spawn exactly 3 agents — not one per story.

## Workflow

### Phase 1: Discover

1. Use the Glob tool to find all files matching `STORIES_GLOB`
2. If `FILENAME_FILTER` is provided and non-empty, filter the file list to only include files whose name contains that substring
3. Read each YAML file and parse both the `setup` array (optional) and `stories` array
4. If a file fails to parse, log a warning and skip it
5. Organize stories by source file — keep each file's stories grouped together (do NOT flatten into a single list)
6. Collect all unique `auth_file` values referenced by stories across all files (deduplicated)
7. If no stories are found, report that and stop
8. Generate `RUN_DIR` using Bash:
   ```bash
   RUN_DIR="bowser-qa-test-results/$(date +%Y%m%d_%H%M%S)_$(uuidgen | tr '[:upper:]' '[:lower:]' | head -c 6)"
   ```
9. **Resolve `MAIN_SITE`:**
   - If `MAIN_SITE` was provided as an argument, use it.
   - Otherwise, check if any story or setup entry URL contains `[MAIN_SITE]`. If so, prompt the user with `AskUserQuestion` asking for the base URL (e.g., `http://localhost:3000`).
   - If no URL contains `[MAIN_SITE]`, skip this step (all URLs are already absolute).
   - Strip any trailing `/` from the value.
   - Save `MAIN_SITE` to `{RUN_DIR}/main_site` using the Write tool.
   - Replace all occurrences of `[MAIN_SITE]` in every story and setup entry URL with the resolved value.
10. For each story, build its `SCREENSHOT_PATH`:
    - `{RUN_DIR}/{file-stem}/{slugified-story-name}/`
    - Example: `bowser-qa-test-results/20260210_143022_a1b2c3/athlete-login/activities-page-loads/`
11. Create the `{RUN_DIR}/auth/` directory if any setup entries exist:
    ```bash
    mkdir -p {RUN_DIR}/auth
    ```

### Phase 1.5: Setup (Auth State)

Run this phase only if any stories reference an `auth_file`.

12. Collect all `setup` entries across all parsed YAML files. Deduplicate by `auth_file` path.
13. For each unique setup entry, spawn one `bowser-qa-agent` via the Task tool using this prompt:

```
You are performing a one-time login to save browser auth state for reuse by parallel test agents.

**Login URL:** {setup.url}
**Auth file to save:** {RUN_DIR}/auth/{setup.auth_file}
**Headed:** {HEADED}

**Workflow:**
{setup.workflow}

After completing the workflow, save the browser state:
  playwright-cli state-save {RUN_DIR}/auth/{setup.auth_file}

Instructions:
- Follow the workflow steps sequentially
- Do NOT take screenshots
- After saving the state, confirm the file was written
- Use this exact format for your final line:
  SETUP: {DONE|FAIL} | {setup.auth_file}
```

14. **Wait for ALL setup agents to complete before proceeding.** If any setup agent fails (reports `SETUP: FAIL`), mark all stories that depend on that `auth_file` as FAIL and exclude them from the spawn phase.

### Phase 2: Spawn

15. Count the number of YAML files that have remaining stories. Make **exactly that many** Task tool calls — one call per file. All Task calls go in a **single message** so they run in parallel. Do NOT make one Task call per story.

For each YAML file, build **one** prompt that contains **all** of that file's stories. The agent will execute them sequentially inside a single session.

**Concrete example:** If `athlete-login.yaml` has 4 stories (Homepage, Login flow, Activities page, Settings page), you make **one** Task call with a prompt like this:

```
Execute the following user stories **sequentially** (one after another) and report results for each.

**Source file:** athlete-login.yaml
**Headed:** false
**Vision:** false

---

### Story 1: Homepage loads
**URL:** http://localhost:3000
**Screenshots:** bowser-qa-test-results/20260228_143022_a1b2c3/athlete-login/homepage-loads/

**Workflow:**
Navigate to the homepage
Verify it loads successfully

---

### Story 2: Login flow
**URL:** http://localhost:3000/login
**Screenshots:** bowser-qa-test-results/20260228_143022_a1b2c3/athlete-login/login-flow/

**Pre-step (do this first, before the workflow):**
Load saved auth state: playwright-cli state-load bowser-qa-test-results/20260228_143022_a1b2c3/auth/cameron.json
Then navigate to: http://localhost:3000/login

**Workflow:**
Verify the login page loads
Fill email and password, click login
Wait for redirect to dashboard

Note: Do NOT perform a manual login — auth state is already saved.

---

### Story 3: Activities page
**URL:** http://localhost:3000/activities
**Screenshots:** bowser-qa-test-results/20260228_143022_a1b2c3/athlete-login/activities-page/

**Pre-step (do this first, before the workflow):**
Load saved auth state: playwright-cli state-load bowser-qa-test-results/20260228_143022_a1b2c3/auth/cameron.json
Then navigate to: http://localhost:3000/activities

**Workflow:**
Verify the activities page loads
Verify key widgets are visible

Note: Do NOT perform a manual login — auth state is already saved.

---

### Story 4: Settings page
**URL:** http://localhost:3000/settings
**Screenshots:** bowser-qa-test-results/20260228_143022_a1b2c3/athlete-login/settings-page/

**Workflow:**
Navigate to the settings page
Verify it loads successfully

---

## Instructions

- Execute stories 1 through 4 **in order** — finish each story completely before starting the next
- For each story, follow each step in its workflow sequentially
- Take a screenshot after each significant step
- Save screenshots to the directory listed under each story's **Screenshots** field
- Report each step as PASS or FAIL with a brief explanation
- If a story has a **Pre-step**, execute it before the workflow (load auth state, then navigate)
- If the server returns an error (5xx, connection refused, timeout, page crash), stop the CURRENT story — report it as FAIL, then move on to the NEXT story
- Do NOT attempt to debug, restart, or fix the server

After completing ALL stories, provide a summary using this exact format:

RESULT: Homepage loads | PASS | Steps: 2/2
RESULT: Login flow | FAIL | Steps: 1/3
RESULT: Activities page | PASS | Steps: 2/2
RESULT: Settings page | PASS | Steps: 2/2
TOTAL: 4 stories | 3 passed | 1 failed
```

Follow this exact pattern for each YAML file. Substitute actual story names, URLs, workflows, screenshot paths, and settings. For stories with `auth_file`, include the **Pre-step** block. For stories without `auth_file`, omit it.

### Phase 3: Collect

16. Wait for all file agent messages to arrive
17. Parse each file agent's report to extract per-story results:
    - For each `RESULT:` line: extract story name, PASS/FAIL, and steps (X/Y)
    - The full agent report text (contains details for all stories in that file)
    - Token usage from the `<usage>` block: `total_tokens`, `tool_uses`, `duration_ms`
18. Include any stories that were pre-failed due to setup failure
19. Also collect token usage from any setup agents that ran in Phase 1.5

### Phase 4: Cleanup and Report

20. Delete ephemeral auth state files:
    ```bash
    rm -rf {RUN_DIR}/auth
    ```
21. Compose the full report markdown (see `Report` section below for the format).
22. Write the report to `{RUN_DIR}/report.md` using the Write tool.
23. Print the report to the terminal so the user can see the results inline.

## Report

Compose this markdown, write it to `{RUN_DIR}/report.md`, then also print it to the terminal:

```
# UI Review Summary

**Run:** {current date and time}
**Site:** {MAIN_SITE} (if set)
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

## Token Usage

| Agent              | Tokens     | Tool Uses | Duration |
| ------------------ | ---------- | --------- | -------- |
| Setup: {auth_file} | {tokens}   | {uses}    | {Xs}     |
| {filename}         | {tokens}   | {uses}    | {Xs}     |
| **Total**          | **{total}** | **{total}** | —     |

## Screenshots
All screenshots saved to: `{RUN_DIR}/`
```

Use ✅ ALL PASSED for status only when every story passed. Use ❌ PARTIAL FAILURE when some passed and some failed. Use ❌ ALL FAILED when none passed.

For the Token Usage table:
- Include one row per setup agent and one row per file agent (not per story)
- Parse `total_tokens`, `tool_uses`, and `duration_ms` from the `<usage>` block in each agent's response
- Convert `duration_ms` to seconds for display (e.g., `45s`)
- The **Total** row sums `total_tokens` and `tool_uses` across all agents
- If token usage data is unavailable for an agent (e.g., it crashed), show `—` for that row
