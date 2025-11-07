---
model: claude-sonnet-4-5-20250929
description: Create a git worktree with isolated configuration for parallel development
argument-hint: [branch-name] [port-offset]
allowed-tools: Bash, Read, Write, Edit, Glob, Grep
---

# Purpose

Create a new git worktree in the `trees/` directory with completely isolated configuration for parallel execution. This enables running multiple instances of the observability system simultaneously without port conflicts, database collisions, or hook misconfiguration.

## Variables

```
PROJECT_CWD: . (current working directory - the main project root)
BRANCH_NAME: $1 (required)
PORT_OFFSET: $2 (optional, defaults to auto-calculated based on existing worktrees, starts at 1)
WORKTREE_BASE_DIR: trees/
WORKTREE_DIR: trees/<BRANCH_NAME>
SERVER_BASE_PORT: 4000
CLIENT_BASE_PORT: 5173
SERVER_PORT: 4000 + (PORT_OFFSET * 10)  # First worktree: 4010, Second: 4020, etc.
CLIENT_PORT: 5173 + (PORT_OFFSET * 10)  # First worktree: 5183, Second: 5193, etc.
OPEN_BROWSER_WHEN_COMPLETE: false       # Set to true to auto-open browser after setup

NOTE: Main repo uses ports 4000 and 5173 (no offset)
      Worktrees start at offset 1 to avoid conflicts with main repo
```

## Instructions

- This is a ONE-SHOT command that creates AND starts a worktree automatically
- Creates a fully functional, isolated clone of the codebase in a separate worktree
- Each worktree runs on unique ports to prevent conflicts when running in parallel
- Port offsets start at 1 and increment (1→4010/5183, 2→4020/5193, 3→4030/5203...)
- Main repo preserves default ports 4000/5173 for primary development work
- All environment configuration must be worktree-specific
- Database files are isolated per worktree (each gets its own events.db)
- Hook scripts will send events to the worktree's specific server instance
- Dependencies are installed automatically for each worktree
- After setup, the script automatically starts both server and client services
- The start script kills any existing processes on the target ports before starting
- Services run in the FOREGROUND - Ctrl+C stops both server and client
- Validation ensures the worktree is ready before starting services
- If branch doesn't exist locally, create it from current HEAD
- If branch exists but isn't checked out, create worktree from it
- Provide clear access URLs so user can immediately use the running instance

## Workflow

### 1. Parse and Validate Arguments

- Read BRANCH_NAME from $1, error if missing
- Read PORT_OFFSET from $2 if provided
- If PORT_OFFSET not provided, calculate next available offset:
  - List all existing worktrees: `git worktree list`
  - Check PROJECT_CWD/trees/ directory for existing worktrees
  - Count existing worktrees and use (count + 1) as offset (1, 2, 3, 4...)
  - IMPORTANT: Offset starts at 1 to preserve main repo ports (4000, 5173)
  - First worktree gets offset 1 → ports 4010, 5183
  - Second worktree gets offset 2 → ports 4020, 5193
- Calculate SERVER_PORT and CLIENT_PORT using offset * 10
- Validate branch name format (no spaces, valid git branch name)

### 2. Pre-Creation Validation

- Check if PROJECT_CWD/trees/ directory exists, create if not: `mkdir -p trees`
- Verify trees/ is in PROJECT_CWD/.gitignore (should be there already)
- Check if worktree already exists at WORKTREE_DIR
- Check if branch exists: `git branch --list <BRANCH_NAME>`
  - If branch doesn't exist, will create it in next step
  - If branch exists, will checkout to create worktree
- Check if calculated ports are available:
  - Check SERVER_PORT: `lsof -i :SERVER_PORT` (should return nothing)
  - Check CLIENT_PORT: `lsof -i :CLIENT_PORT` (should return nothing)
  - If ports are in use, error with message to try different offset

### 3. Create Git Worktree

- From PROJECT_CWD, create worktree with: `git worktree add trees/<BRANCH_NAME> <BRANCH_NAME>`
  - If branch doesn't exist, this creates it from HEAD
  - If branch exists, this checks it out in the worktree
  - This creates WORKTREE_DIR at PROJECT_CWD/trees/<BRANCH_NAME>
- Verify worktree was created: `git worktree list | grep trees/<BRANCH_NAME>`
- All subsequent operations will reference WORKTREE_DIR (which is PROJECT_CWD/trees/<BRANCH_NAME>)

### 4. Setup Root Environment File

- Check if root .env exists in main project at PROJECT_CWD/.env
- If PROJECT_CWD/.env exists:
  - Copy it to worktree root: `cp <PROJECT_CWD>/.env <WORKTREE_DIR>/.env`
  - Note: This preserves API keys (ANTHROPIC_API_KEY, OPENAI_API_KEY, etc.)
- If PROJECT_CWD/.env doesn't exist:
  - Copy .env.sample if available: `cp <PROJECT_CWD>/.env.sample <WORKTREE_DIR>/.env`
  - Add warning to report that user needs to configure API keys


### 5. Validation

- Verify directory structure:
  - Confirm WORKTREE_DIR exists
  - Confirm WORKTREE_DIR/.env exists at root
- List worktrees to confirm: `git worktree list`
- Read back the created env files to confirm values are correct

### 6. Report

Follow the Report section format below to provide comprehensive setup information.

## Report

After successful worktree creation, validation, provide a detailed report in the following format:

```
✅ Git Worktree Created and Started Successfully!

📁 Worktree Details:
   Location: trees/<BRANCH_NAME>
   Branch: <BRANCH_NAME>
   Status: 🟢 RUNNING

⚙️  Environment Files:
   ✓ Root .env (WORKTREE_DIR/.env with API keys)


📝 Important Notes:
   • This worktree is completely isolated from the main codebase
   • You can run multiple worktrees simultaneously with different ports

🔄 To Restart This Worktree Later:

   cd trees/<BRANCH_NAME>


🗑️  To Remove This Worktree:

   # Then remove the worktree:
   git worktree remove trees/<BRANCH_NAME>

   # Or force remove if needed:
   git worktree remove trees/<BRANCH_NAME> --force

🎉 Next Steps:
   2. Open Claude Code in this worktree directory
   3. Run commands - events will stream to this isolated instance
   4. Compare side-by-side with other worktrees or main codebase
   5. Each instance maintains its own database and event history
```

If any validation steps failed or warnings occurred, include an additional section:

```
⚠️  Warnings / Action Required:
- <List any warnings or actions the user needs to take>
```
