---
description: Read user answers from the clarification comment, resolve them into the spec, and validate completeness.
---

**Initial User Prompt:** $ARGUMENTS

You are the agent for the speckit.clarify workflow. Your role is to read the user's answers from the clarification comment (find it in the issue body), resolve them into the feature specification, and validate the spec is ready for planning.

**This is NOT an interactive chat.** The user has already answered the questions by editing the clarification comment. Your job is to find their answers, apply them, and update the spec.

## Initial Setup

**FIRST** - Parse the initial user prompt to extract:
- Environment context = $1 (contains repo, branch, issue_number, issue_title)

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
- If spec comment is empty or missing: **TERMINATE** with label `Err:NO_SPEC`
- If clarification comment is empty or missing: **TERMINATE** with label `Err:NO_CLARIFICATION`
- If no user answers found in the clarification comment: **TERMINATE** with label `Err:NO_ANSWERS`
- **On ANY termination/stop**: Add a label to the issue in the format `Err:{Reason}`

**TERMINATE procedure:**
1. Add label `Err:{Reason}` to issue $ISSUE_NUMBER in $REPO
2. Post a comment on the issue explaining the failure
3. Stop execution

## GitHub Labels for Status Tracking

- `speckit:spec_draft` - spec still has unresolved ambiguities
- `speckit:spec_ready` - spec clarified and ready for planning
- Remove any prior `speckit:spec_*` labels before setting a new one

---

## Phase 1: Load Artifacts

**Step 1: Validate Input**
1. Parse $1 to extract $REPO, $ISSUE_NUMBER, $BRANCH, $ISSUE_TITLE
2. Read the issue body from $REPO issue #$ISSUE_NUMBER
3. Parse the reference table to find:
   - The "spec comment" reference - if missing: TERMINATE with `Err:NO_SPEC`
   - The "clarification comment" reference - if missing: TERMINATE with `Err:NO_CLARIFICATION`

**Step 2: Load Comments**
1. Read the spec comment content from the issue
2. Read the clarification comment content from the issue
3. Store both in memory

---

## Phase 2: Parse User Answers

**Step 1: Extract Answers from Clarification Comment**

The clarification comment was posted by `/speckit.specify` with this structure per question:

```markdown
### Q[N]: [Topic]
...
**YOUR ANSWER: ___**
```

For each question (Q1, Q2, Q3, ...):
1. Find the `**YOUR ANSWER:**` line
2. Extract the user's answer (everything after `YOUR ANSWER:` on that line, trimmed)
3. If the answer is still `___` or blank: mark as **unanswered**
4. If the answer is a single letter (A, B, C, etc.): map it to the corresponding option from the table above that question
5. If the answer is free text: use it as-is

**Step 2: Validate Answers**
1. Count total questions vs answered questions
2. If ALL questions are unanswered (`___` or blank): TERMINATE with `Err:NO_ANSWERS`
3. If SOME questions are unanswered: note them as still pending, proceed with answered ones

---

## Phase 3: Apply Answers to Spec

For each answered question:

**Step 1: Ensure Clarifications Section Exists in Spec**
- Add a `## Clarifications` section if missing (after the overview section)
- Under it, create a `### Session YYYY-MM-DD` subheading for today

**Step 2: Record the Q&A**
- Append: `- Q: <question> -> A: <final answer>`

**Step 3: Apply Clarification to Relevant Section**
- Functional ambiguity -> Update or add a bullet in Functional Requirements
- User interaction / actor distinction -> Update User Stories section
- Data shape / entities -> Update Key Entities section
- Non-functional constraint -> Add/modify criteria in Success Criteria or add Non-Functional section
- Edge case / negative flow -> Add under Edge Cases
- Terminology conflict -> Normalize term across spec

**Step 4: Clean Up**
- If clarification invalidates an earlier ambiguous statement, **replace** it (no duplicates)
- Remove the corresponding `[NEEDS CLARIFICATION]` marker from the spec for each resolved question

**Step 5: Save Spec**
- Edit the spec comment on the issue with the updated content

---

## Phase 4: Handle Remaining Ambiguities

After applying all answered questions, scan the spec for remaining `[NEEDS CLARIFICATION]` markers.

**If unanswered questions remain OR new ambiguities found:**
1. **Update the clarification comment** with only the remaining/new questions (same format as original):

   ```markdown
   ## Speckit: Clarification Needed (Updated)

   **Instructions:** Please answer each remaining question below by replacing the `YOUR ANSWER: ___` placeholder. Once done, run `/speckit.clarify` again.

   **Previously Resolved:** [count] questions answered

   ---

   ### Q[N]: [Topic]
   ...
   **YOUR ANSWER: ___**
   ```

2. Keep label `speckit:spec_draft`
3. Workflow stops here - user edits and triggers `/speckit.clarify` again

**If all `[NEEDS CLARIFICATION]` markers are resolved:**
- Proceed to Phase 5

---

## Phase 5: Validation

Run final validation on the updated spec:

- No remaining `[NEEDS CLARIFICATION]` markers
- Clarifications session contains exactly one bullet per resolved answer (no duplicates)
- Updated sections contain no lingering vague placeholders
- No contradictory earlier statements remain
- Markdown structure valid
- Terminology consistency: same canonical term used across all updated sections

If validation fails on non-clarification items:
- Fix them directly (max 3 iterations)
- If still failing, warn user

---

## Phase 6: Completion

**Step 1: Update Label**
- If all `[NEEDS CLARIFICATION]` markers resolved and validation passes:
  - Remove `speckit:spec_draft` label
  - Set `speckit:spec_ready` label
- If unresolved markers remain:
  - Ensure `speckit:spec_draft` label is set

**Step 2: Update Clarification Comment**
- If all questions resolved: edit the clarification comment to mark it as complete:

  ```markdown
  ## Speckit: Clarifications Resolved

  All [count] questions have been answered and applied to the spec.
  **Resolved on:** [DATE]

  [Original Q&A preserved below for reference]
  ---
  [original content]
  ```

**Step 3: Report**
Post a summary comment on the issue:

```markdown
## Speckit: Clarification Complete

**Questions resolved:** [count answered] / [count total]
**Spec Comment:** (link)
**Sections touched:** [list of section names]

**Coverage Summary:**

| Category | Status |
|----------|--------|
| Functional Scope | [Resolved/Clear/Deferred/Outstanding] |
| Domain & Data Model | [...] |
| Interaction & UX | [...] |
| Non-Functional | [...] |
| Integration | [...] |
| Edge Cases | [...] |
| Constraints | [...] |
| Terminology | [...] |

**Next Steps:**
- If spec_ready: Run `/speckit.plan` to create technical implementation plan
- If spec_draft: Answer remaining questions in the clarification comment, then run `/speckit.clarify` again
```

---

## Behavior Rules

- This command does NOT ask questions interactively - it reads answers from the clarification comment
- If no clarification comment exists, instruct user to run `/speckit.specify` first
- If spec comment missing, instruct user to run `/speckit.specify` first
- If the clarification comment has no answered questions (all still `___`), TERMINATE with `Err:NO_ANSWERS` and remind the user to edit the comment first
- Can be run multiple times - each run resolves whatever new answers it finds
- Avoid speculative tech stack questions unless absence blocks functional clarity
- If all markers are already resolved when this command runs, report "No critical ambiguities detected" and set `speckit:spec_ready`
