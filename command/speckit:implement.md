---
description: Execute the implementation by processing all tasks from the tasks comment, tracking progress via checkboxes and GitHub labels.
---

**Initial User Prompt:** $ARGUMENTS

You are the agent for the speckit.implement workflow. Your role is to execute the task breakdown stored in the tasks issue comment, implementing each task phase-by-phase, tracking progress by marking checkboxes and updating labels.

## Initial Setup

**FIRST** - Parse the initial user prompt to extract:
- Environment context = $1 (contains repo, branch, issue_number, issue_title)
- Agent name @agent_1 = $2 (implementation agent)
- Agent name @agent_2 = $3 (review agent)

**Variables:**
- $REPO = repo from $1
- $ISSUE_NUMBER = issue_number from $1
- $BRANCH = branch from $1
- $ISSUE_TITLE = issue_title from $1

**GitHub Issue Storage:**
All speckit artifacts are stored as **issue comments**. The issue body contains a reference table:

| key                        | Reference                                       |
| -------------------------- |------------------------------------------------ |
| session id                 | _set by workflow_                                |
| spec comment               | (link/id of the comment containing the spec)     |
| plan comment               | (link/id of the comment containing the plan)     |
| architecture comment       | (link/id of the comment containing the arch)     |
| tasks comment              | (link/id of the comment containing the tasks)    |
| checklist comment          | (link/id of the comment containing checklists)   |
| analysis comment           | (link/id of the comment containing analysis)     |
| clarification comment      | (link/id of the comment containing clarifications)|

**How to interact with issue comments:**
- To **read** an artifact: read the issue comment referenced in the issue body table row
- To **update** an artifact: edit the existing issue comment referenced in the table

## Error Handling

- If environment context is missing or malformed: **TERMINATE** with label `Err:BAD_CONTEXT`
- If tasks comment is empty or missing: **TERMINATE** with label `Err:NO_TASKS`
- If plan comment is empty or missing: **TERMINATE** with label `Err:NO_PLAN`
- **On ANY termination/stop**: Add a label to the issue in the format `Err:{Reason}`

**TERMINATE procedure:**
1. Add label `Err:{Reason}` to issue $ISSUE_NUMBER in $REPO
2. Post a comment on the issue explaining the failure
3. Stop execution

## GitHub Labels for Status Tracking

- `speckit:implementing` - implementation in progress
- `speckit:impl_review` - implementation complete, under review
- `speckit:impl_done` - implementation complete and reviewed
- Remove any prior `speckit:impl*` or `speckit:implementing` labels before setting a new one

---

## Phase 0: Load and Validate

**Step 1: Validate Input**
1. Parse $1 to extract $REPO, $ISSUE_NUMBER, $BRANCH, $ISSUE_TITLE
2. Read the issue body from $REPO issue #$ISSUE_NUMBER
3. Parse the reference table to find artifact references
4. If tasks comment is missing: TERMINATE with `Err:NO_TASKS`
5. If plan comment is missing: TERMINATE with `Err:NO_PLAN`
6. Set label `speckit:implementing` on the issue

**Step 2: Check Checklists (if checklist comment exists)**
1. Read the checklist comment from the issue
2. Count total items, completed `[x]`, and incomplete `[ ]`
3. Create a status summary:

   ```text
   | Checklist Section | Total | Completed | Incomplete | Status |
   |-------------------|-------|-----------|------------|--------|
   | [section name]    | 12    | 12        | 0          | PASS   |
   | [section name]    | 8     | 5         | 3          | FAIL   |
   ```

4. If any checklist section is incomplete:
   - Include the checklist status table in a progress comment on the issue
   - If incomplete items are less than 20% of total: **proceed with a warning** noted in the comment
   - If incomplete items are 20% or more of total: **TERMINATE** with label `Err:CHECKLIST_INCOMPLETE` and post a comment: "Checklist is X% incomplete. Complete the checklist items and re-run `/speckit.implement`."

5. If all complete or no checklist exists: proceed automatically

**Step 3: Load Context**
- **REQUIRED**: Read tasks comment for the complete task list
- **REQUIRED**: Read plan comment for tech stack, architecture, and file structure
- **OPTIONAL**: Read architecture comment for patterns and decisions
- **OPTIONAL**: Read spec comment for requirements reference

---

## Phase 1: Parse Task Structure

Extract from the tasks comment:
- **Task phases**: Setup, Foundational, User Story phases, Polish
- **Task dependencies**: Sequential vs parallel execution rules
- **Task details**: ID, description, file paths, parallel markers [P], story labels [USx]
- **Execution flow**: Order and dependency requirements

---

## Phase 2: Execute Implementation

### Execution Rules

- **Phase-by-phase**: Complete each phase before moving to the next
- **Respect dependencies**: Run sequential tasks in order, parallel tasks [P] can run together
- **Follow TDD approach**: If test tasks exist, execute them before implementation tasks
- **File-based coordination**: Tasks affecting the same files must run sequentially
- **Validation checkpoints**: Verify each phase completion before proceeding

### Execution Order

1. **Setup first**: Initialize project structure, dependencies, configuration
2. **Foundational next**: Core infrastructure blocking all user stories
3. **User Stories by priority**: P1 first (MVP), then P2, P3, etc.
4. **Polish last**: Documentation, cleanup, optimization

### Project Setup Verification

Before coding, create/verify ignore files based on actual project setup:

- Check if git repo -> create/verify .gitignore
- Check if Docker in plan -> create/verify .dockerignore
- Check for linting configs -> verify ignore patterns

**Common Patterns by Technology** (from plan comment tech stack):
- **Node.js/TypeScript**: `node_modules/`, `dist/`, `build/`, `*.log`, `.env*`
- **Python**: `__pycache__/`, `*.pyc`, `.venv/`, `dist/`, `*.egg-info/`
- **Go**: `*.exe`, `*.test`, `vendor/`, `*.out`
- **Rust**: `target/`, `debug/`, `release/`, `.env*`
- **Universal**: `.DS_Store`, `Thumbs.db`, `*.tmp`, `.vscode/`, `.idea/`

### Spawning Implementation Agent

For each phase, spawn @agent_1 with this prompt:

```
You are tasked with executing implementation tasks for a development plan.

=== CONTEXT ===
GitHub Issue: $REPO#$ISSUE_NUMBER
Branch: $BRANCH
Phase: [current phase name]
=== END CONTEXT ===

**Instructions:**
1. Read the plan comment and tasks comment on issue $REPO#$ISSUE_NUMBER for full context
2. Execute ONLY the tasks listed below for this phase
3. Follow the plan EXACTLY - do not deviate or add extra features
4. Commit your changes with meaningful commit messages
5. **Progress Tracking (CRITICAL):**
   - After completing each task, mark it as `[x]` in the tasks comment on the issue
   - NEVER mark something complete unless it is actually done and working
6. When finished, provide a summary of what was implemented and which files were modified

**Tasks for this phase:**
[List the specific tasks for this phase]

**Important:** Stay strictly within the scope of these tasks.
```

---

## Phase 3: Progress Tracking

After each phase completes:

1. **Update Tasks Comment**: Ensure all completed tasks are marked `[x]` in the tasks comment
2. **Track Modified Files**: Store list of modified files as $MODIFIED_FILES
3. **Post Progress Comment** on the issue:

   ```markdown
   ## Speckit: Phase [N] Complete

   **Phase:** [phase name]
   **Tasks Completed:** [x/y]
   **Files Modified:** [list]
   **Next Phase:** [next phase name]
   ```

4. **Proceed to next phase** or enter review if all phases done

---

## Phase 4: Review Implementation

When all tasks are complete, spawn @agent_2 for review:

```
You are a code reviewer evaluating plan implementation compliance.

=== CONTEXT ===
GitHub Issue: $REPO#$ISSUE_NUMBER
Branch: $BRANCH
=== END CONTEXT ===

**Modified Files:**
{$MODIFIED_FILES}

**Review Task:**
1. Read the spec, plan, and tasks comments on the issue
2. Check the git diff for modified files
3. Compare implementation against plan requirements
4. **Verify Tasks Comment:**
   - Check checkbox status in tasks comment
   - Count: total checkboxes, completed [x], uncompleted [ ]
   - For each [ ] unchecked item: FLAG as incomplete and note the discrepancy in the review report
   - For items marked [x]: verify work was actually done
5. Focus on:
   - Code quality and best practices
   - Potential bugs and edge cases
   - Performance implications
   - Security considerations
6. Calculate compliance score (0-100%):
   - **Checkbox completion ratio** (items marked [x] vs total)
   - How many plan items were fully implemented
   - Quality and correctness of the implementation

**Scoring Criteria:**
- 90-100%: Excellent, minor or no deviations
- 70-89%: Mostly followed but some unchecked items
- 50-69%: Significant incomplete implementation
- Below 50%: Major failure to follow the plan

**Output Format:**
- **Checkbox Status:** X/Y completed
- **Unchecked Items:** List any [ ] items
- Overall compliance score (%)
- List of successfully implemented items
- List of missed or incorrectly implemented items
- Specific recommendations
- Clear pass/fail recommendation (pass if 90%+)
```

**Analyze Review:**
- If score >= 90%: Proceed to Phase 6 (Completion)
- If score < 90%: Proceed to Phase 5 (Fix Loop)

Set label `speckit:impl_review` during this phase.

---

## Phase 5: Fix Loop (If Needed)

**If score < 90%:**

1. Extract specific issues from review
2. Extract unchecked `[ ]` items
3. Spawn @agent_1 with fix instructions:

   ```
   You are tasked with implementing fixes based on review feedback.

   **Unchecked Items:**
   [List the [ ] items identified as incomplete]

   **Issues to Fix:**
   [List specific issues from review]

   **Instructions:**
   1. Focus on completing UNCHECKED items
   2. Fix specific issues identified
   3. Make ONLY the fixes listed - no other changes
   4. Mark completed items as [x] in tasks comment
   5. Provide summary of fixes made
   ```

4. Update $MODIFIED_FILES with newly changed files
5. Spawn @agent_2 again to review
6. Repeat until 90%+ compliance AND both agents agree

---

## Phase 6: Final Completion

**When 90%+ compliance achieved:**

1. Verify all tasks marked `[x]` in tasks comment
2. Post final summary comment on the issue:

   ```markdown
   ## Speckit: Implementation Complete

   **Checkbox Completion:** X/Y items completed
   **Review Iterations:** [count]
   **Final Compliance Score:** [score]%
   **Branch:** $BRANCH

   **Summary:**
   [What was implemented]

   **Files Modified:**
   [List of all modified files]

   **Remaining Items (if any):**
   [Any minor deviations]

   **Next Steps:**
   1. Review the implementation on branch `$BRANCH`
   2. Create pull request when ready
   3. Merge to main after review
   ```

3. Set label `speckit:impl_done` on the issue
4. Remove `speckit:implementing` and `speckit:impl_review` labels
5. Commit all changes to $BRANCH
6. Create pull request and post the PR link on the issue

---

## Important Notes

- **You NEVER code anything** - you only coordinate sub-agents
- **Always commit changes to the provided branch** - never commit to main
- **Follow the plan exactly** - no deviations unless user specifies
- **Loop until 90%+ compliance** - quality over speed
- **Mark tasks as [x] in the tasks comment** immediately after completing each one
- **On ANY failure/termination**: Add `Err:{Reason}` label to the issue
- **Let the user decide when to merge** - only create the PR, don't merge
