---
model: claude-sonnet-4-5-20250929
description: List all git worktrees with their configuration and status
allowed-tools: Bash, Read, Glob, Grep
---

# Purpose

List all git worktrees in the `trees/` directory with comprehensive information including branch names, directories, environment variables.

## Variables

```
PROJECT_CWD: . (current working directory - the main project root)
WORKTREE_BASE_DIR: trees/
```

## Instructions

- List all worktrees managed by git
- For each worktree in trees/, gather configuration details
- Display comprehensive information in a clear, organized format
- Show which worktrees are active vs stopped
- Provide quick action commands for each worktree

## Workflow

### 1. List Git Worktrees

- Run: `git worktree list`
- Parse output to identify all worktrees
- Filter for worktrees in PROJECT_CWD/trees/ directory
- Extract:
  - Worktree path
  - Branch name
  - Commit hash (if available)

### 2. Gather Configuration for Each Worktree

For each worktree found in trees/:

**Extract Branch/Directory Info:**
- Worktree directory: `trees/<branch-name>`
- Branch name from git worktree list
- Working directory path


**Read Root Configuration:**
- Check if `<worktree>/.env` exists
- Note presence/absence (contains API keys, don't display values)

### 3. Report

Follow the Report section format below.

## Report

After gathering all information, provide a comprehensive report in the following format:

```
📊 Git Worktrees Overview

═══════════════════════════════════════════════════════════════

📈 Summary:
   Total Worktrees: <count>

═══════════════════════════════════════════════════════════════

🌳 Main Repository (Default)
   📁 Location: <project-root>
   🌿 Branch: <current-branch>

───────────────────────────────────────────────────────────────

🌳 Worktree: <branch-name>
   📁 Location: trees/<branch-name>
   🌿 Branch: <branch-name>
   📝 Commit: <commit-hash-short>

───────────────────────────────────────────────────────────────

[Repeat for each worktree]

═══════════════════════════════════════════════════════════════

💡 Quick Commands:

Create new worktree:
└─ /create_worktree <branch-name> 

Remove worktree:
└─ /remove_worktree <branch-name>

View this list again:
└─ /list_worktrees

═══════════════════════════════════════════════════════════════
```

If no worktrees exist in trees/:

```
📊 Git Worktrees Overview

═══════════════════════════════════════════════════════════════

🌳 Main Repository (Default)
   📁 Location: <project-root>
   🌿 Branch: <current-branch>

═══════════════════════════════════════════════════════════════

ℹ️  No worktrees found in trees/ directory

💡 Create your first worktree:
   /create_worktree <branch-name>

   This will:
   • Create isolated git worktree
   • Install dependencies
   • Start services automatically

═══════════════════════════════════════════════════════════════
```

If worktrees have configuration issues:

```
⚠️  Configuration Warnings:

• trees/<branch-name>: Missing .env files
  └─ Fix: Recreate with /create_worktree <branch-name>

• trees/<branch-name>: Dependencies not installed
  └─ Fix: cd trees/<branch-name>/apps/server && bun install
  └─ Fix: cd trees/<branch-name>/apps/client && bun install

• trees/<branch-name>: Services running but ports mismatch
  └─ Fix: Stop services and update .env files
```

## Notes

- Main repository is always shown first (uses default ports)
- Worktrees are sorted alphabetically by branch name
- Service status is checked in real-time
- Orphaned worktrees (in git but not in trees/) are noted
- PIDs are shown for running processes for easy termination
- All commands are copy-paste ready
