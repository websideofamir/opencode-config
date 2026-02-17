---
description: Auto planner - takes raw task and plans the execution
---

**Initial User Prompt:** $ARGUMENTS

You're the main agent for the Auto Planner workflow. Your role is to coordinate sub-agents to execute comprehensive planning and verify completion. You do NOT code anything yourself - you can ONLY delegate tasks to sub-agents.

## Global Configuration
- **MAX_RETRIES**: 5 (for each phase)
- **MAX_REFINEMENT_ITERATIONS**: 5 (total for Phase 4)
- **CURRENT_ITERATION**: 0 (global counter)

## Initial Setup:
**FIRST**
- Environment context = $1 (contains repo, branch, issue_number, issue_title)
- @agent_1 = $2
- @agent_2 = $3

**GitHub Issue Storage:**
All artifacts are stored as **issue comments** instead of files. The issue body contains a reference table:

| key                        | Refrence                                                |
| -------------------------- |---------------------------------------------------------|
| session id                 | _set by workflow_                                       |
| task comment               | (link/id of the comment containing the task if exists)  |
| plan comment               | (link/id of the comment containing the plan if exists)  |
| architecture comment       | (link/id of the comment containing the arch if exists)  |

**Variables:**
- $REPO = repo from $1
- $ISSUE_NUMBER = issue_number from $1
- $BRANCH = branch from $1
- $ISSUE_TITLE = issue_title from $1

**How to interact with issue comments:**
- To **read** an artifact: read the issue comment referenced in the issue body table row
- To **create** an artifact: post a new comment on the issue, then update the issue body table row with the comment URL
- To **update** an artifact: edit the existing issue comment referenced in the table

## Error Handling:
- If task comment is empty or missing: **TERMINATE** with label `Err:Task comment not found`
- If any phase exceeds MAX_RETRIES: **TERMINATE** with label `Err:Phase failed after {MAX_RETRIES} attempts`
- If refinement exceeds MAX_REFINEMENT_ITERATIONS: **PROCEED** to completion with best effort
- **On ANY termination/stop**: Add a label to the issue in the format `Err:{Reason}` describing why the workflow stopped

**TERMINATE procedure:**
1. Add label `Err:{Reason}` to issue $ISSUE_NUMBER in $REPO
2. Post a comment on the issue explaining the failure
3. Stop execution

**Editing comments:**
You can use gh tool to directly edit the comments if they are not provided via an mcp

---

## Phase 0: Setup & Context Extraction

**Step 1: Validate Input**
1. Parse $1 to extract $REPO, $ISSUE_NUMBER, $BRANCH, $ISSUE_TITLE
2. Read the issue body from $REPO issue #$ISSUE_NUMBER
3. Parse the reference table to find the "task comment" reference
4. If task comment reference is empty or missing: TERMINATE with `Err:NO_TASK`

**Step 2: Extract Task Context**
1. Read the task comment from the issue
2. Extract and store:
   - Task ID and title (use $ISSUE_TITLE if not in comment)
   - Parent feature ID (if exists) → $FEATURE_ID
   - Parent feature title (if exists) → $FEATURE_TITLE
   - Key requirements (list)
   - Expected outcome
   - Dependencies
3. Store as $TASK_CONTEXT (max 5000 tokens)

**Step 3: Extract Feature Architecture (if exists)**
If task references parent feature architecture:
1. Read feature architecture from referenced comment if exists
2. Extract ONLY:
   - Technology stack decisions
   - Key architectural patterns
   - Critical constraints (security, performance)
   - Integration requirements
3. Store as $FEATURE_ARCH_SUMMARY (max 12000 tokens)
4. Store reference as $FEATURE_ARCH_REF

**Step 4: Prepare Inline Context**
```
=== TASK CONTEXT ===
{$TASK_CONTEXT}

=== FEATURE ARCHITECTURE (from {$FEATURE_ARCH_REF}) ===
{$FEATURE_ARCH_SUMMARY}

Full details available at: {$FEATURE_ARCH_PATH} comment on issue {$REPO} {$ISSUE_NUMBER}
```

---

## Phase 1: Architectural Analysis

**RETRY_COUNT**: 0

**Step 1: Spawn architect agent**

Send this prompt to architect agent:

```
You're creating an architectural analysis for a development task.

=== CONTEXT PROVIDED ===
{$INLINE_CONTEXT}
=== END CONTEXT ===

**GitHub Issue:** $REPO#$ISSUE_NUMBER

**Instructions:**
1. OPTIONAL: Read the task comment on the issue if you need more details
2. OPTIONAL: Read feature architecture at $FEATURE_ARCH_REF if available
3. If feature architecture exists, REUSE its technology decisions
4. ONLY research task-specific patterns (not broad architectural topics)
5. Produce the architectural analysis as markdown content (DO NOT write to a file)
6. Include these sections:
   - Context Analysis
   - Technology Recommendations
   - System Architecture
   - Integration Patterns
   - Implementation Guidance
7. Mark critical decisions as IMPORTANT
8. DO NOT write implementation code
9. Return the full analysis content as your response
```

**Step 2: Save & Verify Output**
1. Wait for architect completion
2. Check if the "architecture comment" row in the issue body already has a reference:
   - If yes: **edit** that existing comment with the new analysis content
   - If no: **post** a new comment on the issue with the analysis content, then update the issue body table "architecture comment" row with the comment URL
3. If comment was saved successfully: Proceed to Phase 2
4. If save failed:
   - Increment RETRY_COUNT
   - If RETRY_COUNT < MAX_RETRIES: Restart Phase 1
   - If RETRY_COUNT >= MAX_RETRIES: TERMINATE with `Err:Architecture creation failed`

---

## Phase 2: Implementation Planning

**RETRY_COUNT**: 0

**Step 1: Spawn @agent_1**

Send this prompt to @agent_1:

```
You're creating a detailed implementation plan based on architectural analysis.

=== CONTEXT PROVIDED ===
{$INLINE_CONTEXT}
=== END CONTEXT ===

**GitHub Issue:** $REPO#$ISSUE_NUMBER

**Architecture:** Read the architecture comment on the issue (referenced in the issue body table)

**Instructions:**
1. Read the architectural analysis from the architecture comment
2. OPTIONAL: Read the task comment if needed
3. Produce the implementation plan as markdown content (DO NOT write to a file)
4. Include these sections:
   - Implementation Overview
   - Component Details
   - Data Structures
   - API Design
   - Testing Strategy
   - Development Phases
   - **Progress Checklist** (REQUIRED - see format below)
5. Follow architectural guidelines marked as IMPORTANT
6. DO NOT write executable code
7. Include only illustrative code snippets
8. Return the full plan content as your response
9. **Progress Checklist Format:**
   - Use markdown checkboxes for ALL major implementation items
   - Think like a senior dev's todo list - major milestones, not micro-tasks
   - Support one level of nesting for logical grouping
   - Example format:
     ```markdown
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
     ```
   - This checklist will be used during execution to track progress
```

**Step 2: Save & Verify Output**
1. Wait for @agent_1 completion
2. Check if the "plan comment" row in the issue body already has a reference:
   - If yes: **edit** that existing comment with the new plan content
   - If no: **post** a new comment on the issue with the plan content, then update the issue body table "plan comment" row with the comment URL
3. If comment was saved successfully: Proceed to Phase 3
4. If save failed:
   - Increment RETRY_COUNT
   - If RETRY_COUNT < MAX_RETRIES: Restart Phase 2
   - If RETRY_COUNT >= MAX_RETRIES: TERMINATE with `Err:PLAN_AILED`

---

## Phase 3: Review

**Step 1: Spawn @agent_2**

Send this prompt to @agent_2:

```
You're reviewing an implementation plan for feasibility and architectural alignment.

=== CONTEXT PROVIDED ===
{$INLINE_CONTEXT}
=== END CONTEXT ===

**GitHub Issue:** $REPO#$ISSUE_NUMBER

**Instructions:**
1. Read the architecture comment on the issue (referenced in the issue body table)
2. Read the plan comment on the issue (referenced in the issue body table)
3. OPTIONAL: Read the task comment if needed
4. Evaluate using these criteria:
   - **Implementation Feasibility (40%)**: Is the plan detailed enough? Does it follow guidelines?
   - **Architectural Alignment (30%)**: Does it align with architectural decisions?
   - **Completeness (20%)**: Are all requirements covered? Testing strategy?
   - **Integration Quality (10%)**: Proper integration with prior tasks?

5. Provide your review in this format:

**Overall Score:** XX%

**Implementation Score:** XX% (40% weight)
- [Specific feedback on implementation approach]

**Architectural Score:** XX% (30% weight)
- [Specific feedback on architectural alignment]

**Completeness Score:** XX% (20% weight)
- [Specific feedback on requirements coverage]

**Integration Score:** XX% (10% weight)
- [Specific feedback on integration quality]

**Critical Issues:** [Must-fix items, if any]

**Recommendations:** [Specific improvements needed]

**Final Decision:**
- If 90%+: "PASS - Ready for implementation"
- If below 90%: "NEEDS REFINEMENT - [brief summary of issues]"
```

**Step 2: Analyze Review**
1. Receive review from @agent_2
2. Store the complete review as $REVIEW_FEEDBACK
3. Extract overall score as $OVERALL_SCORE
4. If $OVERALL_SCORE >= 90: Proceed to Phase 5
5. If $OVERALL_SCORE < 90: Proceed to Phase 4

---

## Phase 4: Refinement Loop

**REFINEMENT_COUNT**: 0

**Step 1: Categorize Issues**
From $REVIEW_FEEDBACK, extract:
- Architectural issues (if any)
- Implementation issues (if any)

**Step 2: Refine Based on Issues**

**If Architectural Issues Exist:**
Spawn @agent_2 with this prompt:

```
You need to fix architectural issues based on review feedback.

=== CONTEXT PROVIDED ===
{$INLINE_CONTEXT}
=== END CONTEXT ===

**GitHub Issue:** $REPO#$ISSUE_NUMBER

**Review Feedback:**
{$REVIEW_FEEDBACK}

**Instructions:**
1. Read the current architecture comment on the issue
2. Fix the architectural issues mentioned in the feedback
3. Return the updated architectural analysis as your response (DO NOT write to a file)
4. Make ONLY the necessary fixes
5. Mark updated decisions as IMPORTANT
```

After receiving the updated architecture, **edit** the existing architecture comment on the issue with the new content.

**If Implementation Issues Exist:**
Spawn @agent_1 with this prompt:

```
You need to fix implementation issues based on review feedback.

=== CONTEXT PROVIDED ===
{$INLINE_CONTEXT}
=== END CONTEXT ===

**GitHub Issue:** $REPO#$ISSUE_NUMBER

**Review Feedback:**
{$REVIEW_FEEDBACK}

**Instructions:**
1. Read the current plan comment on the issue
2. Fix the implementation issues mentioned in the feedback
3. Return the updated implementation plan as your response (DO NOT write to a file)
4. Make ONLY the necessary fixes
5. Ensure plan follows architectural guidelines
```

After receiving the updated plan, **edit** the existing plan comment on the issue with the new content using `gh`.

**Step 3: Re-evaluate**
1. Wait for refinement completion
2. Increment REFINEMENT_COUNT
3. If REFINEMENT_COUNT < MAX_REFINEMENT_ITERATIONS:
   - Return to Phase 3 for re-review
4. If REFINEMENT_COUNT >= MAX_REFINEMENT_ITERATIONS:
   - Proceed to Phase 5 (best effort completion)
   - Note: "Maximum refinement iterations reached, proceeding with current quality"

---

## Phase 5: Final Completion

**Step 1: Generate Summary**
Post a comment on the issue with this summary:

```
## Auto Plan Workflow Complete

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
- Iterations required: {Total count}

**Next Steps:**
1. Review the architecture and plan comments on this issue
2. Begin implementation following the plan
3. Commit changes when ready

**Workflow Benefits:**
- Research-backed architectural decisions
- Comprehensive implementation planning
- Dynamic agent selection
- Token-optimized processing
- Quality-based assessment
```

**Step 2: Final Instructions**
- set task ai:plan_ready to the issue labels if workflow was successfule
- Indicate workflow is complete
- Do NOT auto-commit

---

## Error Recovery Procedures

### **Phase Failures**
- **Phase 1 Failure**: TERMINATE with label `Err:ARCH`
- **Phase 2 Failure**: TERMINATE with label `Err:PLAN`
- **Phase 3 Failure**: TERMINATE with label `Err:REVIEW`

### **Recovery Options**
1. **Retry with different agents**: Suggest trying different @agent_1/@agent_2 combinations
2. **Manual intervention**: User can edit issue comments manually
3. **Partial completion**: Proceed with available artifacts if acceptable

### **Debug Information**
Always log:
- Current phase and iteration count
- Issue comment references being used
- Agent responses
- Error messages

---

## Token Optimization Features Maintained

1. **Inline Context Passing**: Context extracted once, passed to all agents
2. **Conditional Research**: Agents only research what's necessary
3. **Efficient Comment Reading**: Optional comment reading based on need
4. **Focused Prompts**: Clear, specific instructions to reduce back-and-forth

This rewritten flow eliminates infinite loops, provides clear error handling, and maintains all optimization benefits while being more reliable and easier to debug.
