# Issue Response Validation

Validate GitHub issues, enhance unclear descriptions, and ensure all answers are captured in the issue description (source of truth).

## Input
- **Issue ID**: $ARGUMENTS (GitHub issue number, e.g., "47")

## Core Principle
**The issue description is the source of truth.** All decisions, answers, and clarifications from comments MUST be incorporated into the main issue description. Comments are for discussion; the description is the final specification.

## Instructions

1. **Fetch the issue with labels and all comments**
   ```bash
   gh issue view $ARGUMENTS --json title,body,labels,comments,author
   ```

2. **Check if the issue has labels**
   - If labels array is empty, suggest an appropriate label based on issue content
   - Common labels: `enhancement`, `bug`, `documentation`, `refactor`, `feature`, `chore`, `performance`
   - Add the suggested label:
     ```bash
     gh issue edit $ARGUMENTS --add-label "suggested-label"
     ```

3. **Assess issue quality** - If the issue is new or lacks detail:
   - **Title**: If vague or unclear, suggest a more descriptive title
     ```bash
     gh issue edit $ARGUMENTS --title "New clearer title"
     ```
   - **Description**: Enhance with:
     - Clear problem statement or feature description
     - Acceptance criteria (if missing)
     - Technical context from codebase investigation
     - Structured sections (Description, Current State, Proposed Solution, Tasks, etc.)
   - Add a comment noting: "Enhanced issue title/description for clarity"

4. **Identify all questions** in the issue body and comments that were asked by the maintainer/author

5. **Check each comment** for answers provided by the user (issue owner)

6. **For each question, determine if it has been answered:**
   - Mark as answered if a clear response was provided
   - Mark as unanswered/unclear if no response or ambiguous answer

7. **Update the issue description immediately** when answers are found:
   - DO NOT leave answers only in comments
   - Incorporate every answered question into the main description
   - Add/update a "Decisions" section with resolved items
   - Update relevant sections (requirements, acceptance criteria, etc.)
   - Use clear formatting (tables, bullet points) for readability

8. **If questions remain unanswered:**
   - Create a comment listing the unanswered questions
   - Ask for clarification on any ambiguous answers

9. **Add a summary comment** that includes:
   - If title/description was enhanced: "Enhanced issue title/description for clarity"
   - If a label was added: "Added label: `label-name`"
   - If answers were incorporated: "Updated issue description with X decisions from comments"
   - Any pending questions or items

## Updating the Issue Description

When updating, use this command:
```bash
gh issue edit $ARGUMENTS --body "updated body content"
```

Structure the description with clear sections:
- **Description**: What and why
- **Current State**: How it works now (if applicable)
- **Decisions**: All resolved questions/answers from discussion
- **Implementation Tasks**: Checklist of work items
- **Acceptance Criteria**: How to verify completion

## Output Format

Provide a summary:
- Issue quality: (enhanced title/description / already clear)
- Label status: (existing: X / added: Y)
- Total questions identified: X
- Questions answered: Y
- Questions pending: Z
- Description updates: (X decisions incorporated)
- Actions taken: (updated issue / added label / enhanced description / added comment)
