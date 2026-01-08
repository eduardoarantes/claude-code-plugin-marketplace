---
description: Execute task implementation with TDD using preparation, execution, and review phases
---

Instructions: $ARGUMENTS

## Language Detection

Before starting the implementation workflow, determine the programming language from the task context:

1. **Check the GitHub Issue**: Use `gh issue view` on the INPROGRESS issue to read the issue body and comments for language specifications
2. **Check the Architecture Wiki**: Read the associated architecture documentation (`Issue-{issue_number}-{Feature-Name}-Architecture`) for tech stack details
3. **If not specified**, ask the user which language to use for this task

Note: A project can have multiple languages - determine the language specific to THIS task.

## Agent Selection

Based on the determined language, use the appropriate plugin agents:

**TypeScript/JavaScript** (use `typescript-coding-agents` plugin):
- Preparation: `typescript-coding-agents:task-prep-architect`
- Execution: `typescript-coding-agents:task-executor-tdd`
- Review: `typescript-coding-agents:task-implementation-reviewer`

**Python** (use `python-coding-agents` plugin):
- Preparation: `python-coding-agents:task-prep-architect`
- Execution: `python-coding-agents:task-executor-tdd`
- Review: `python-coding-agents:task-implementation-reviewer`

## Implementation Phases

1. **Preparation Phase**: YOU MUST Spawn the language-appropriate task-prep-architect agent using the Task tool to analyze requirements, plan approach, and set up the implementation strategy
2. **Execution Phase**: YOU MUST Spawn the language-appropriate task-executor-tdd agent using the Task tool to perform the actual implementation work using test-driven development
3. **Review Phase**: YOU MUST Spawn the language-appropriate task-implementation-reviewer agent using the Task tool to evaluate the implementation quality and completeness

**MANDATORY EXECUTION RULES - NEVER SKIP THESE - ABSOLUTELY NON-NEGOTIABLE:**
- You must enforce strict sequential execution - no phase can begin until the previous phase is completely finished
- If the review phase identifies issues requiring rework, you must restart the cycle from preparation through execution to review again


**WORKFLOW MANAGEMENT RULES - ABSOLUTELY NON-NEGOTIABLE - MUST NEVER BE VIOLATED:**
- ALWAYS start with preparation phase using task-prep-architect - NEVER SKIP THIS PHASE
- ALWAYS wait for complete confirmation from each agent before proceeding to the next phase
- If review identifies defects or incomplete work, IMMEDIATELY cycle back to preparation for rework planning
- Continue the preparation-execution-review cycle until the review agent confirms successful completion
- Maintain clear communication about which phase is active and why transitions occur
- Track the number of cycles to identify potential infinite loops and escalate if needed

**Quality Assurance:**
- Ensure each phase agent receives complete context from previous phases
- Verify that rework addresses specific issues identified in reviews
- Confirm that all project-specific requirements from CLAUDE.md are considered throughout the process
- Document the rationale for any cycle restarts

**Communication Protocol:**
- Clearly announce each phase transition
- Summarize key outputs from each completed phase
- Explain the reasoning when cycling back for rework
- Provide status updates on overall implementation progress

Run them sequentially: execute the language-appropriate task-prep-architect agent first. After task-prep-architect completes, run the language-appropriate task-executor-tdd agent, and when it completes run the language-appropriate task-implementation-reviewer agent - no parallelization
