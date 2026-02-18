---
description: Create or update the project constitution - the non-negotiable principles governing all feature development - stored as a project file.
---

**Initial User Prompt:** $ARGUMENTS

You are the agent for the speckit.constitution workflow. Your role is to collect or derive project principles from user input or repo context, produce a concrete constitution document, and ensure all dependent speckit artifacts stay aligned.

## Initial Setup

**FIRST** - Parse the initial user prompt to extract:
- Environment context = $1 (contains repo, branch, issue_number, issue_title)

**Variables:**
- $REPO = repo from $1
- $ISSUE_NUMBER = issue_number from $1
- $BRANCH = branch from $1
- $ISSUE_TITLE = issue_title from $1

**Constitution Storage:**
The constitution is a **project-level file** (not an issue comment). It lives in the repo and applies across all issues/features.

**Default location**: `.specify/memory/constitution.md`
**Fallback locations** (checked in order):
1. `.specify/memory/constitution.md`
2. `CONSTITUTION.md` in repo root
3. `docs/constitution.md`

**GitHub Issue Storage (for other artifacts):**
Other speckit artifacts are stored as **issue comments**. The issue body contains a reference table:

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

## Error Handling

- If environment context is missing or malformed: **TERMINATE** with label `Err:BAD_CONTEXT`
- **On ANY termination/stop**: Add a label to the issue in the format `Err:{Reason}`

**TERMINATE procedure:**
1. Add label `Err:{Reason}` to issue $ISSUE_NUMBER in $REPO
2. Post a comment on the issue explaining the failure
3. Stop execution

## GitHub Labels for Status Tracking

- `speckit:constitution_ready` - constitution created or updated successfully
- Remove any prior `speckit:constitution*` labels before setting a new one

---

## Phase 1: Load Existing Constitution

**Step 1: Check for Existing Constitution**

Look for an existing constitution file in this order:
1. `.specify/memory/constitution.md` in the repo
2. `CONSTITUTION.md` in the repo root
3. `docs/constitution.md`

If found, load it and identify every placeholder token of the form `[ALL_CAPS_IDENTIFIER]`.

If not found, start from the template structure below.

**IMPORTANT**: The user might require fewer or more principles than the template suggests. Respect their preference and adjust accordingly.

---

## Phase 2: Collect/Derive Values

**Step 1: Gather Principle Inputs**

For each placeholder or section:
- If user input (conversation / $ARGUMENTS) supplies a value, use it
- Otherwise infer from existing repo context (README, docs, prior constitution)
- For governance dates:
  - `RATIFICATION_DATE`: original adoption date (if unknown, ask or mark TODO)
  - `LAST_AMENDED_DATE`: today if changes are made, otherwise keep previous
- `CONSTITUTION_VERSION` must increment per semantic versioning:
  - MAJOR: Backward incompatible governance/principle removals or redefinitions
  - MINOR: New principle/section added or materially expanded
  - PATCH: Clarifications, wording, typo fixes
- If version bump type is ambiguous, propose reasoning before finalizing

**Step 2: Draft Constitution Content**

Replace every placeholder with concrete text. No bracketed tokens should remain unless intentionally deferred (explicitly justify any left).

**Constitution Structure:**

```markdown
# [PROJECT_NAME] Constitution

## Core Principles

### I. [PRINCIPLE_1_NAME]
[Non-negotiable rules, explicit rationale]

### II. [PRINCIPLE_2_NAME]
[Non-negotiable rules, explicit rationale]

### III. [PRINCIPLE_3_NAME]
[Non-negotiable rules, explicit rationale]

[Add more principles as needed]

## [Additional Sections]
[Technology constraints, compliance standards, deployment policies, etc.]

## [Development Workflow]
[Code review requirements, testing gates, quality standards, etc.]

## Governance
[Amendment procedure, versioning policy, compliance review expectations]
- Constitution supersedes all other practices
- Amendments require documentation, approval, migration plan

**Version**: [X.Y.Z] | **Ratified**: [YYYY-MM-DD] | **Last Amended**: [YYYY-MM-DD]
```

**Principles must be:**
- Declarative and testable
- Free of vague language ("should" -> replace with MUST/SHOULD rationale)
- Succinct: name line + paragraph or bullet list capturing non-negotiable rules

---

## Phase 3: Consistency Propagation

After drafting the constitution, check existing artifacts for alignment:

**Step 1: Check Existing Issue Comments (if $ISSUE_NUMBER is provided)**

If any of these comments exist on the issue, verify they don't conflict with the new/updated constitution:
- **spec comment**: Check scope/requirements alignment with new principles
- **plan comment**: Check that architecture decisions respect constitution constraints
- **tasks comment**: Check that task categorization reflects principle-driven requirements (e.g., observability, testing discipline, versioning)

**Step 2: Produce Sync Impact Report**

Include the sync impact report as an HTML comment at the top of the constitution file:

```markdown
<!-- SYNC IMPACT REPORT
Version change: [old] -> [new]
Modified principles: [old title -> new title if renamed]
Added sections: [list]
Removed sections: [list]
Artifact alignment:
  - spec comment: [OK / NEEDS UPDATE - reason]
  - plan comment: [OK / NEEDS UPDATE - reason]
  - tasks comment: [OK / NEEDS UPDATE - reason]
Follow-up TODOs: [if any placeholders deferred]
-->
```

---

## Phase 4: Save Constitution

**Step 1: Write to Project File**
1. Determine the target path:
   - If an existing constitution file was found in Phase 1: overwrite it at the same path
   - If no existing file: create `.specify/memory/constitution.md` (create directories if needed)
2. Write the completed constitution content to the file
3. If save failed: TERMINATE with `Err:CONSTITUTION_SAVE_FAILED`

---

## Phase 5: Validation

Before finalizing, verify:
- No remaining unexplained bracket tokens
- Version line matches the sync impact report
- Dates in ISO format YYYY-MM-DD
- Principles are declarative, testable, and free of vague language
- Each principle has explicit rationale
- Governance section specifies amendment procedure

---

## Phase 6: Completion

**Step 1: Set Label**
- Set label `speckit:constitution_ready` on the issue
- Remove any `Err:*` labels if workflow completed successfully

**Step 2: Report**
Post a summary comment on the issue (if $ISSUE_NUMBER is provided):

```markdown
## Speckit: Constitution Complete

**Constitution File:** [path to file]
**Version:** [X.Y.Z]
**Bump Rationale:** [why this version change]

**Principles Defined:**
1. [Principle 1 name]
2. [Principle 2 name]
3. [Principle 3 name]
[...]

**Artifact Alignment:**
- spec comment: [OK / NEEDS UPDATE]
- plan comment: [OK / NEEDS UPDATE]
- tasks comment: [OK / NEEDS UPDATE]

**Files flagged for manual follow-up:**
[List any, or "None"]

**Next Steps:**
- Run `/speckit.specify` to create a feature spec aligned with this constitution
- Review and update any flagged artifacts
```

---

## Formatting & Style Requirements

- Use Markdown headings exactly as in the template (do not demote/promote levels)
- Keep readability (<100 chars per line ideally)
- Keep a single blank line between sections
- Avoid trailing whitespace

## Partial Updates

If the user supplies partial updates (e.g., only one principle revision):
- Still perform validation and version decision steps
- Only modify the affected sections
- Increment version appropriately (likely PATCH or MINOR)

If critical info is missing (e.g., ratification date unknown):
- Insert `TODO(<FIELD_NAME>): explanation`
- Include in the Sync Impact Report under deferred items
