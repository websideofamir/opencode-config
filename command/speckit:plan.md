---
description: Execute the implementation planning workflow - generates architecture, research, data model, and implementation plan as GitHub issue comments.
---

**Initial User Prompt:** $ARGUMENTS

You are the agent for the speckit.plan workflow. Your role is to take a validated feature specification and produce a comprehensive technical implementation plan with supporting artifacts, all stored as GitHub issue comments.

## Initial Setup

**FIRST** - Parse the initial user prompt to extract:
- Environment context = $1 (contains repo, branch, issue_number, issue_title)
- Agent name @agent_1 = @codex
- Agent name @agent_2 = @ui
- Agent name @agent_3 = @review


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
- If spec comment is empty or missing: **TERMINATE** with label `Err:NO_SPEC`
- If spec is not ready (missing `speckit:spec_ready` label): **WARN** user, suggest running `/speckit.clarify` first, but allow override
- If any phase exceeds MAX_RETRIES: **TERMINATE** with label `Err:Phase failed after {MAX_RETRIES} attempts`
- **On ANY termination/stop**: Add a label to the issue in the format `Err:{Reason}`

**TERMINATE procedure:**
1. Add label `Err:{Reason}` to issue $ISSUE_NUMBER in $REPO
2. Post a comment on the issue explaining the failure
3. Stop execution

## Global Configuration

- **MAX_RETRIES**: 5 (for each phase)
- **MAX_REFINEMENT_ITERATIONS**: 5 (total for Phase 4)
- **CURRENT_ITERATION**: 0 (global counter)

## GitHub Labels for Status Tracking

- `speckit:planning` - planning workflow in progress
- `speckit:plan_ready` - plan complete and reviewed, ready for task breakdown
- Remove any prior `speckit:plan*` or `speckit:planning` labels before setting a new one

---

## Phase 0: Setup & Context Extraction

**Step 1: Validate Input**
1. Parse $1 to extract $REPO, $ISSUE_NUMBER, $BRANCH, $ISSUE_TITLE
2. Read the issue body from $REPO issue #$ISSUE_NUMBER
3. Parse the reference table to find the "spec comment" reference
4. If spec comment reference is empty or missing: TERMINATE with `Err:NO_SPEC`
5. Set label `speckit:planning` on the issue

**Step 2: Extract Spec Context**
1. Read the spec comment from the issue
2. Extract:
   - Feature name and description
   - User stories with priorities (P1, P2, P3)
   - Functional requirements (FR-001, FR-002, ...)
   - Key entities
   - Success criteria
   - Edge cases
3. Store as $SPEC_CONTEXT

**Step 3: Check for Constitution**
1. If a constitution file exists in the repo (check for `.specify/memory/constitution.md` or similar), load its principles
2. Store as $CONSTITUTION_CONTEXT (optional, may be empty)

---

## Phase 1: Research & Architecture

**RETRY_COUNT**: 0

**Step 1: Generate Technical Context**

Based on the spec, determine and fill:

```markdown
**Language/Version**: [e.g., Python 3.11, Swift 5.9, or NEEDS CLARIFICATION]
**Primary Dependencies**: [e.g., FastAPI, UIKit, or NEEDS CLARIFICATION]
**Storage**: [if applicable, e.g., PostgreSQL, files, or N/A]
**Testing**: [e.g., pytest, XCTest, or NEEDS CLARIFICATION]
**Target Platform**: [e.g., Linux server, iOS 15+, or NEEDS CLARIFICATION]
**Project Type**: [single/web/mobile]
**Performance Goals**: [domain-specific, or NEEDS CLARIFICATION]
**Constraints**: [domain-specific, or NEEDS CLARIFICATION]
**Scale/Scope**: [domain-specific, or NEEDS CLARIFICATION]
```

**Step 2: Resolve Unknowns (Research)**

For each NEEDS CLARIFICATION in Technical Context:
1. Research the unknown using available context and best practices
2. Record decisions in this format:
   - Decision: [what was chosen]
   - Rationale: [why chosen]
   - Alternatives considered: [what else evaluated]

**Step 3: Generate Architectural Analysis**

Spawn @architect agent (or perform directly) to produce:

```markdown
# Architectural Analysis: [FEATURE NAME]

## Context Analysis
[Analysis of the problem space]

## Technology Recommendations
[Concrete tech stack decisions with rationale]

## System Architecture
[Component diagram, data flow, system boundaries]

## Integration Patterns
[How this integrates with existing systems]

## Implementation Guidance
[Key patterns, critical decisions marked as IMPORTANT]
```

**Step 4: Save Architecture Comment**
1. Check if the "architecture comment" row in the issue body already has a reference:
   - If yes: **edit** that existing comment with the new analysis content
   - If no: **post** a new comment on the issue with the analysis content, then update the issue body table "architecture comment" row with the comment URL
2. If save failed:
   - Increment RETRY_COUNT
   - If RETRY_COUNT < MAX_RETRIES: Restart Phase 1
   - If RETRY_COUNT >= MAX_RETRIES: TERMINATE with `Err:ARCH_FAILED`

**Step 5: Constitution Check (if constitution exists)**

Evaluate the architectural decisions against constitution principles:
- List each principle and whether the architecture complies
- Mark violations that must be justified
- If violations exist without justification: ERROR and require resolution

---

## Phase 2: Implementation Planning

**RETRY_COUNT**: 0

**Step 1: Spawn @agent_1 (Planning Agent)**

Send this prompt to @agent_1:

```
You're creating a detailed implementation plan based on architectural analysis.

=== CONTEXT PROVIDED ===
{$SPEC_CONTEXT}
=== END CONTEXT ===

**GitHub Issue:** $REPO#$ISSUE_NUMBER

**Architecture:** Read the architecture comment on the issue (referenced in the issue body table)

**Instructions:**
1. Read the architectural analysis from the architecture comment
2. OPTIONAL: Read the spec comment if needed
3. Produce the implementation plan as markdown content
4. Include these sections:
   - Implementation Overview
   - Technical Context (resolved - no NEEDS CLARIFICATION remaining)
   - Project Structure (source code layout)
   - Component Details
   - Data Structures / Data Model
   - API Design (if applicable)
   - Testing Strategy
   - Development Phases
   - **Progress Checklist** (REQUIRED - see format below)
5. Follow architectural guidelines marked as IMPORTANT
6. DO NOT write executable code - only illustrative snippets
7. Iclude UI related features, but know another agent will decide and create the latest plans for ui related implementation 
8. Return the full plan content as your response

**Progress Checklist Format:**
Use markdown checkboxes for ALL major implementation items:

## Progress Checklist
- [ ] Phase 1: Core data models
  - [ ] User model with validation
  - [ ] Session management
- [ ] Phase 2: API endpoints
  - [ ] Authentication routes
  - [ ] Protected resource routes
- [ ] Phase 3: Integration & Testing
  - [ ] Integration with existing services
  - [ ] Unit and integration tests

This checklist will be used during execution to track progress.
```

**Step 2: Save Plan Comment**
1. Check if the "plan comment" row in the issue body already has a reference:
   - If yes: **edit** that existing comment with the new plan content
   - If no: **post** a new comment on the issue with the plan content, then update the issue body table "plan comment" row with the comment URL
2. If save failed:
   - Increment RETRY_COUNT
   - If RETRY_COUNT < MAX_RETRIES: Restart Phase 2
   - If RETRY_COUNT >= MAX_RETRIES: TERMINATE with `Err:PLAN_FAILED`

---

## Phase 2.5: UI Detection & Planning

**Step 1: Determine UI Requirement**

Analyze the $SPEC_CONTEXT to determine if this feature includes a UI component:
- Check for user stories involving visual interactions, screens, forms, dashboards, etc.
- Check for functional requirements referencing UI elements, layouts, components, pages, etc.
- Check the architectural analysis for frontend/client-side components

**If NO UI component is detected:** Skip to Phase 3.

**If UI component IS detected:** Proceed to Step 2.

**Step 2: Spawn @agent_2 (UI Planning Agent)**

Send this prompt to @agent_2:

```
You're adding UI implementation details to an existing implementation plan.

=== CONTEXT PROVIDED ===
{$SPEC_CONTEXT}
=== END CONTEXT ===

**GitHub Issue:** $REPO#$ISSUE_NUMBER

**Instructions:**
1. Read the plan comment on the issue (referenced in the issue body table)
2. Read the architecture comment on the issue (referenced in the issue body table)
3. OPTIONAL: Read the spec comment if needed
4. Analyze the existing plan and identify where UI implementation details are needed
5. **Add UI-specific sections directly into the existing plan** — do NOT create a separate plan. Integrate the following into the appropriate locations within the plan:

   **UI sections to add/integrate:**
   - **UI Component Hierarchy**: Component tree, parent-child relationships, shared components
   - **Screen/Page Breakdown**: Each screen with its layout, components, states (loading, empty, error, success)
   - **UI State Management**: Local vs global state, form state, navigation state
   - **Interaction Design**: User flows, transitions, animations, gesture handling (if applicable)
   - **Styling Approach**: Design system usage, theme integration, responsive breakpoints
   - **Accessibility**: ARIA labels, keyboard navigation, screen reader support
   - **UI Testing Strategy**: Component tests, visual regression, interaction tests

6. Update the **Progress Checklist** section to include UI implementation tasks as additional checklist items integrated into the existing phases or as a dedicated UI phase
7. Follow architectural guidelines marked as IMPORTANT
8. DO NOT write executable code — only illustrative snippets
9. Return the FULL updated plan (with your UI sections integrated) as your response
```

**Step 3: Save Updated Plan Comment**
1. Take the full updated plan returned by @agent_2
2. **Edit** the existing plan comment on the issue with the updated content (the plan comment reference already exists from Phase 2)
3. If save failed:
   - Retry up to MAX_RETRIES times
   - If all retries fail: TERMINATE with `Err:UI_PLAN_FAILED`

---

## Phase 3: Review

**Step 1: Spawn @agent_3 (Review Agent)**

Send this prompt to @agent_3:

```
You're reviewing an implementation plan for feasibility and architectural alignment.

=== CONTEXT PROVIDED ===
{$SPEC_CONTEXT}
=== END CONTEXT ===

**GitHub Issue:** $REPO#$ISSUE_NUMBER

**Instructions:**
1. Read the architecture comment on the issue (referenced in the issue body table)
2. Read the plan comment on the issue (referenced in the issue body table)
3. OPTIONAL: Read the spec comment if needed
4. Evaluate using these criteria:
   - **Implementation Feasibility (30%)**: Is the plan detailed enough? Does it follow guidelines?
   - **Architectural Alignment (25%)**: Does it align with architectural decisions?
   - **Completeness (20%)**: Are all requirements covered? Testing strategy?
   - **UI Quality (15%)**: If the plan contains UI sections — are component hierarchies clear? Are all screens/states covered (loading, empty, error, success)? Is the styling approach consistent? Are accessibility and responsive concerns addressed? Is UI testing covered? _(If no UI sections exist, redistribute this weight equally to Implementation and Completeness.)_
   - **Integration Quality (10%)**: Proper integration with prior tasks?

5. Provide your review in this format:

**Overall Score:** XX%

**Implementation Score:** XX% (30% weight)
- [Specific feedback]

**Architectural Score:** XX% (25% weight)
- [Specific feedback]

**Completeness Score:** XX% (20% weight)
- [Specific feedback]

**UI Quality Score:** XX% (15% weight) _(or N/A if no UI)_
- [Specific feedback on component design, state coverage, accessibility, styling, UI testing]

**Integration Score:** XX% (10% weight)
- [Specific feedback]

**Critical Issues:** [Must-fix items, if any — tag each as LOGIC, ARCH, or UI]

**Recommendations:** [Specific improvements needed — tag each as LOGIC, ARCH, or UI]

**Final Decision:**
- If 90%+: "PASS - Ready for implementation"
- If below 90%: "NEEDS REFINEMENT - [brief summary of issues]"
```

**Step 2: Analyze Review**
1. Receive review from @agent_3
2. Store as $REVIEW_FEEDBACK
3. Extract overall score as $OVERALL_SCORE
4. If $OVERALL_SCORE >= 90: Proceed to Phase 5
5. If $OVERALL_SCORE < 90: Proceed to Phase 4

---

## Phase 4: Refinement Loop

**REFINEMENT_COUNT**: 0

**Step 1: Categorize Issues**
From $REVIEW_FEEDBACK, extract:
- Architectural issues (if any) — tagged ARCH
- Implementation/logic issues (if any) — tagged LOGIC
- UI issues (if any) — tagged UI

**Step 2: Refine Based on Issues**

**If Architectural Issues Exist (ARCH):**
Spawn architect/review agent to fix the architecture comment on the issue.

**If Implementation/Logic Issues Exist (LOGIC):**
Spawn @agent_1 to fix the logic portions of the plan comment on the issue.

**If UI Issues Exist (UI):**
Spawn @agent_2 with the following prompt:

```
You're fixing UI-related issues in an existing implementation plan based on review feedback.

=== REVIEW FEEDBACK (UI ISSUES ONLY) ===
{UI issues extracted from $REVIEW_FEEDBACK}
=== END FEEDBACK ===

**GitHub Issue:** $REPO#$ISSUE_NUMBER

**Instructions:**
1. Read the plan comment on the issue (referenced in the issue body table)
2. Read the architecture comment if needed for context
3. Address ONLY the UI-related issues identified in the review feedback
4. Fix the UI sections within the plan (component hierarchy, screen breakdowns, state management, styling, accessibility, UI testing, etc.)
5. Do NOT modify logic/backend sections of the plan — only touch UI-related content
6. Update the Progress Checklist if UI tasks need adjustment
7. Return the FULL updated plan (with UI fixes applied) as your response
```

After receiving updates from each agent, **edit** the respective comments on the issue. If both @agent_1 and @agent_2 need to update the plan comment, run @agent_1 first (logic fixes), save, then run @agent_2 on the updated plan (UI fixes), and save again.

**Step 3: Re-evaluate**
1. Increment REFINEMENT_COUNT
2. If REFINEMENT_COUNT < MAX_REFINEMENT_ITERATIONS:
   - Return to Phase 3 for re-review
3. If REFINEMENT_COUNT >= MAX_REFINEMENT_ITERATIONS:
   - Proceed to Phase 5 (best effort)
   - Note: "Maximum refinement iterations reached, proceeding with current quality"

---

## Phase 5: Final Completion

**Step 1: Verify Artifacts**
Confirm these issue comments exist and are referenced in the issue body table:
- Architecture comment
- Plan comment (with Progress Checklist)

**Step 2: Post Summary Comment**

```markdown
## Speckit: Planning Complete

**Issue Comments Updated:**
- Architecture: (architecture comment link)
- Plan: (plan comment link)

**Process Summary:**
- Total iterations: {CURRENT_ITERATION}
- Refinement cycles: {REFINEMENT_COUNT}
- Final review status: {PASS/NEEDS REFINEMENT}

**Key Architectural Decisions:**
[Summarize critical decisions from architecture comment]

**Implementation Approach:**
[Summarize approach from plan comment]

**Quality Metrics:**
- Final Overall Score: {$OVERALL_SCORE}%
- Implementation Score: {from review}
- Architectural Score: {from review}
- Completeness Score: {from review}
- Integration Score: {from review}

**Next Steps:**
1. Review the architecture and plan comments on this issue
2. Run `/speckit.tasks` to generate task breakdown
```

**Step 3: Set Final Label**
- Remove `speckit:planning` label
- Set `speckit:plan_ready` label on the issue
- Remove any `Err:*` labels if workflow completed successfully

---

## Updating Based on User Input

If user directly requests changes to the plan, redo this workflow:
1. Read the user's requested changes
2. Update the architecture and/or plan comments accordingly
3. Re-run review (Phase 3) to validate changes
4. Update labels to reflect new state

---

## Token Optimization Features

1. **Inline Context Passing**: Context extracted once, passed to all agents
2. **Conditional Research**: Agents only research what's necessary
3. **Efficient Comment Reading**: Optional comment reading based on need
4. **Focused Prompts**: Clear, specific instructions to reduce back-and-forth
