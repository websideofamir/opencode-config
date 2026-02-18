---
description: Create or update the feature specification from a natural language feature description, stored as a GitHub issue comment.
---

**Initial User Prompt:** $ARGUMENTS

You are the agent for the speckit.specify workflow. Your role is to transform a natural language feature description into a structured, testable feature specification stored as a GitHub issue comment.

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
- To **create** an artifact: post a new comment on the issue, then update the issue body table row with the comment URL
- To **update** an artifact: edit the existing issue comment referenced in the table

## Error Handling

- If environment context is missing or malformed: **TERMINATE** with label `Err:BAD_CONTEXT`
- If feature description is empty: **TERMINATE** with label `Err:NO_DESCRIPTION`
- **On ANY termination/stop**: Add a label to the issue in the format `Err:{Reason}` describing why the workflow stopped

**TERMINATE procedure:**
1. Add label `Err:{Reason}` to issue $ISSUE_NUMBER in $REPO
2. Post a comment on the issue explaining the failure
3. Stop execution

## GitHub Labels for Status Tracking

This command uses labels to communicate state:
- `speckit:spec_draft` - specification created but has [NEEDS CLARIFICATION] markers
- `speckit:spec_ready` - specification complete and validated, ready for next phase
- Remove any prior `speckit:spec_*` labels before setting a new one

user prompt: $2
---

## Phase 1: Parse Feature Description

**Step 1: Validate Input**
1. Parse $1 to extract $REPO, $ISSUE_NUMBER, $BRANCH, $ISSUE_TITLE
2. Read the issue body from $REPO issue #$ISSUE_NUMBER
3. Extract the feature description from either:
   - The user prompt if provided
   - The issue body/title if no arguments given
4. If no feature description can be found: TERMINATE with `Err:NO_DESCRIPTION`

**Step 2: Extract Key Concepts**
From the feature description, identify:
- Actors (who uses this)
- Actions (what they do)
- Data (what entities are involved)
- Constraints (any limitations mentioned)

---

## Phase 2: Generate Feature Specification

Follow this execution flow:

1. **Parse user description**
   - If empty: TERMINATE with `Err:NO_DESCRIPTION`

2. **Extract key concepts** from description
   - Identify: actors, actions, data, constraints

3. **For unclear aspects:**
   - Make informed guesses based on context and industry standards
   - Only mark with `[NEEDS CLARIFICATION: specific question]` if:
     - The choice significantly impacts feature scope or user experience
     - Multiple reasonable interpretations exist with different implications
     - No reasonable default exists
   - **LIMIT: Maximum 3 [NEEDS CLARIFICATION] markers total**
   - Prioritize clarifications by impact: scope > security/privacy > user experience > technical details

4. **Fill User Scenarios & Testing section**
   - Prioritize as user journeys ordered by importance (P1, P2, P3)
   - Each user story must be independently testable
   - Include acceptance scenarios in Given/When/Then format

5. **Generate Functional Requirements**
   - Each requirement must be testable
   - Use reasonable defaults for unspecified details

6. **Define Success Criteria**
   - Create measurable, technology-agnostic outcomes
   - Include both quantitative and qualitative measures
   - Each criterion must be verifiable without implementation details

7. **Identify Key Entities** (if data involved)

**Specification Content Structure:**

```markdown
# Feature Specification: [FEATURE NAME]

**Feature Branch**: `$BRANCH`
**Created**: [DATE]
**Status**: Draft
**Input**: User description: "$ARGUMENTS"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - [Brief Title] (Priority: P1)

[Describe this user journey in plain language]

**Why this priority**: [Explain the value]

**Independent Test**: [How to test independently]

**Acceptance Scenarios**:

1. **Given** [initial state], **When** [action], **Then** [expected outcome]

---

### User Story 2 - [Brief Title] (Priority: P2)
[...]

---

### Edge Cases

- What happens when [boundary condition]?
- How does system handle [error scenario]?

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST [specific capability]
- **FR-002**: System MUST [specific capability]

### Key Entities *(include if feature involves data)*

- **[Entity 1]**: [What it represents, key attributes]
- **[Entity 2]**: [What it represents, relationships]

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: [Measurable metric]
- **SC-002**: [Measurable metric]
```

---

## Phase 3: Store Specification as Issue Comment

**Step 1: Post or Update Comment**
1. Check if the "spec comment" row in the issue body already has a reference:
   - If yes: **edit** that existing comment with the new spec content
   - If no: **post** a new comment on the issue with the spec content, then update the issue body table "spec comment" row with the comment URL
2. If save failed: TERMINATE with `Err:SPEC_SAVE_FAILED`

---

## Phase 4: Specification Quality Validation

After writing the spec comment, validate against quality criteria:

**Quality Checklist (run internally):**

- [ ] No implementation details (languages, frameworks, APIs)
- [ ] Focused on user value and business needs
- [ ] Written for non-technical stakeholders
- [ ] All mandatory sections completed
- [ ] Requirements are testable and unambiguous
- [ ] Success criteria are measurable
- [ ] Success criteria are technology-agnostic
- [ ] All acceptance scenarios are defined
- [ ] Edge cases are identified
- [ ] Scope is clearly bounded

**Validation Handling:**

- **If all items pass and no [NEEDS CLARIFICATION] markers**:
  - Set label `speckit:spec_ready` on the issue
  - Proceed to completion

- **If items fail (excluding [NEEDS CLARIFICATION])**:
  1. List the failing items and specific issues
  2. Update the spec comment to address each issue
  3. Re-run validation until all items pass (max 3 iterations)
  4. If still failing after 3 iterations, warn user and set label `speckit:spec_draft`

- **If [NEEDS CLARIFICATION] markers remain**:
  1. Set label `speckit:spec_draft` on the issue
  2. Extract all `[NEEDS CLARIFICATION: ...]` markers from the spec (max 3)
  3. **Post a clarification comment** on the issue with ALL questions formatted for the user to edit inline. Use this format:

     ```markdown
     ## Speckit: Clarification Needed

     **Instructions:** Please answer each question below by replacing the `YOUR ANSWER: ___` placeholder with your choice (option letter or custom answer). Once you have answered all questions, run `/speckit.clarify` to apply your answers to the spec.

     ---

     ### Q1: [Topic]

     **Context**: [Quote relevant spec section]

     **What we need to know**: [Specific question from NEEDS CLARIFICATION marker]

     **Recommended:** Option [X] - [reasoning why this is the best choice]

     | Option | Answer | Implications |
     |--------|--------|--------------|
     | A      | [First answer] | [Implication] |
     | B      | [Second answer] | [Implication] |
     | C      | [Third answer] | [Implication] |

     **YOUR ANSWER: ___**

     ---

     ### Q2: [Topic]
     [Same format as Q1]

     **YOUR ANSWER: ___**

     ---

     ### Q3: [Topic]
     [Same format as Q1, if needed]

     **YOUR ANSWER: ___**
     ```

  4. **Save the clarification comment reference** in the issue body table under "clarification comment"
  5. The workflow **STOPS here** - the user will edit the comment with their answers and then trigger `/speckit.clarify` to continue

---

## Phase 5: Completion

**Step 1: Report**
Post a summary comment on the issue:

```markdown
## Speckit: Specification Complete

**Status:** [READY / DRAFT]
**Spec Comment:** (link to spec comment)
**Branch:** $BRANCH

**Sections Completed:**
- User Scenarios: [count] stories defined
- Requirements: [count] functional requirements
- Success Criteria: [count] measurable outcomes

**Clarifications:** [count remaining / all resolved]

**Next Steps:**
- If DRAFT: Answer the questions in the clarification comment (link to comment), then run `/speckit.clarify` to resolve them
- If READY: Run `/speckit.plan` to create technical implementation plan
```

**Step 2: Set Final Label**
- If spec is complete and validated: ensure `speckit:spec_ready` label is set
- If spec has unresolved clarifications: ensure `speckit:spec_draft` label is set
- Remove any `Err:*` labels if workflow completed successfully

---

## General Guidelines

### Focus
- Focus on **WHAT** users need and **WHY**
- Avoid HOW to implement (no tech stack, APIs, code structure)
- Written for business stakeholders, not developers

### Section Requirements
- **Mandatory sections**: Must be completed for every feature
- **Optional sections**: Include only when relevant
- When a section doesn't apply, remove it entirely (don't leave as "N/A")

### For AI Generation
1. **Make informed guesses**: Use context, industry standards, and common patterns
2. **Document assumptions**: Record reasonable defaults
3. **Limit clarifications**: Maximum 3 [NEEDS CLARIFICATION] markers
4. **Prioritize clarifications**: scope > security/privacy > user experience > technical details
5. **Think like a tester**: Every vague requirement should fail the "testable and unambiguous" check

### Success Criteria Guidelines
Success criteria must be:
1. **Measurable**: Include specific metrics (time, percentage, count, rate)
2. **Technology-agnostic**: No mention of frameworks, languages, databases, or tools
3. **User-focused**: Describe outcomes from user/business perspective
4. **Verifiable**: Can be tested without knowing implementation details

**Good examples:**
- "Users can complete checkout in under 3 minutes"
- "System supports 10,000 concurrent users"

**Bad examples (implementation-focused):**
- "API response time is under 200ms"
- "React components render efficiently"
