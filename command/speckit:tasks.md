---
description: Generate an actionable, dependency-ordered task breakdown for the feature based on spec and plan, stored as a GitHub issue comment.
---

**Initial User Prompt:** $ARGUMENTS

You are the agent for the speckit.tasks workflow. Your role is to take the spec and plan artifacts from issue comments and generate a structured, dependency-ordered task list organized by user story, stored back as an issue comment.

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
- If spec comment is empty or missing: **TERMINATE** with label `Err:NO_SPEC`
- If plan comment is empty or missing: **TERMINATE** with label `Err:NO_PLAN`
- **On ANY termination/stop**: Add a label to the issue in the format `Err:{Reason}`

**TERMINATE procedure:**
1. Add label `Err:{Reason}` to issue $ISSUE_NUMBER in $REPO
2. Post a comment on the issue explaining the failure
3. Stop execution

## GitHub Labels for Status Tracking

- `speckit:tasks_ready` - task breakdown complete and ready for implementation
- Remove any prior `speckit:tasks*` labels before setting a new one

---

## Phase 1: Load Design Artifacts

**Step 1: Validate Input**
1. Parse $1 to extract $REPO, $ISSUE_NUMBER, $BRANCH, $ISSUE_TITLE
2. Read the issue body from $REPO issue #$ISSUE_NUMBER
3. Parse the reference table to find artifact references
4. If spec comment reference is empty or missing: TERMINATE with `Err:NO_SPEC`
5. If plan comment reference is empty or missing: TERMINATE with `Err:NO_PLAN`

**Step 2: Load Artifacts**
Read from issue comments:
- **Required**: plan comment (tech stack, libraries, structure), spec comment (user stories with priorities)
- **Optional**: architecture comment (architectural decisions, patterns)
- Note: Not all issues have all artifacts. Generate tasks based on what's available.

---

## Phase 2: Extract Task Sources

**From Spec Comment:**
- Extract user stories with their priorities (P1, P2, P3, etc.)
- Extract functional requirements (FR-001, FR-002, ...)
- Extract edge cases
- Extract success criteria

**From Plan Comment:**
- Extract tech stack and libraries
- Extract project structure / file layout
- Extract component details
- Extract data model / entities
- Extract API design (if applicable)
- Extract testing strategy
- Extract development phases
- **Extract UI sections (if present):**
  - UI component hierarchy
  - Screen/page breakdown
  - UI state management approach
  - Styling approach and design system
  - Animations and transitions
  - Accessibility requirements
  - UI testing strategy

**From Architecture Comment (if available):**
- Extract key architectural patterns
- Extract integration requirements
- Extract critical constraints

---

## Phase 3: Generate Task Breakdown

### Task Format (REQUIRED)

Every task MUST strictly follow this format:

```text
- [ ] [TaskID] [P?] [Story?] Description with file path
```

**Format Components:**
1. **Checkbox**: ALWAYS start with `- [ ]` (markdown checkbox)
2. **Task ID**: Sequential number (T001, T002, T003...) in execution order
3. **[P] marker**: Include ONLY if task is parallelizable (different files, no dependencies)
4. **[UI] marker**: Include if this task is a UI task (templates, components, views, pages, layouts, styles, animations). Logic tasks do NOT get this marker.
5. **[Story] label**: REQUIRED for user story phase tasks only
   - Format: [US1], [US2], [US3], etc. (maps to user stories from spec)
   - Setup phase: NO story label
   - Foundational phase: NO story label
   - User Story phases: MUST have story label
   - Polish phase: NO story label
6. **Description**: Clear action with exact file path

**Examples:**
- `- [ ] T001 Create project structure per implementation plan`
- `- [ ] T005 [P] Implement authentication middleware in src/middleware/auth.py`
- `- [ ] T012 [P] [US1] Create User model in src/models/user.py`
- `- [ ] T014 [US1] Implement UserService in src/services/user_service.py`
- `- [ ] T015 [P] [UI] [US1] Create LoginForm component in src/components/LoginForm.tsx`
- `- [ ] T016 [UI] [US1] Implement login page layout and styles in src/pages/Login.tsx`
- `- [ ] T017 [UI] [US1] Add form validation animations in src/components/LoginForm.tsx`

### Task Organization

1. **From User Stories (spec)** - PRIMARY ORGANIZATION:
   - Each user story (P1, P2, P3...) gets its own phase
   - Map all related components to their story: models, services, endpoints, tests
   - Mark story dependencies (most stories should be independent)

2. **From Plan/Architecture:**
   - Map each entity to the user story(ies) that need it
   - If entity serves multiple stories: put in earliest story or Setup phase
   - Map endpoints to their user stories

3. **From Setup/Infrastructure:**
   - Shared infrastructure -> Setup phase (Phase 1)
   - Foundational/blocking tasks -> Foundational phase (Phase 2)
   - Story-specific setup -> within that story's phase

### Phase Structure

Generate the task comment with this structure:

```markdown
# Tasks: [FEATURE NAME]

**Prerequisites**: spec comment (required), plan comment (required), architecture comment (optional)
**Tests**: [OPTIONAL - only include if explicitly requested in spec]

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2)

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization and basic structure

- [ ] T001 Create project structure per implementation plan
- [ ] T002 Initialize [language] project with [framework] dependencies
- [ ] T003 [P] Configure linting and formatting tools

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story

**CRITICAL**: No user story work can begin until this phase is complete

- [ ] T004 [task based on plan]
- [ ] T005 [P] [task based on plan]

**Checkpoint**: Foundation ready - user story implementation can begin

---

## Phase 3: User Story 1 - [Title] (Priority: P1) MVP

**Goal**: [Brief description]
**Independent Test**: [How to verify this story works alone]

### Logic Implementation for User Story 1

- [ ] T010 [P] [US1] Create [Entity] model in src/models/[entity].py
- [ ] T011 [US1] Implement [Service] in src/services/[service].py
- [ ] T012 [US1] Implement [endpoint] in src/[location]/[file].py

### UI Implementation for User Story 1

- [ ] T013 [P] [UI] [US1] Create [Component] in src/components/[Component].tsx
- [ ] T014 [UI] [US1] Implement [Page] layout and styles in src/pages/[Page].tsx
- [ ] T015 [UI] [US1] Wire [Component] to [Service] and add interaction states

**Checkpoint**: User Story 1 fully functional and testable independently

---

## Phase 4: User Story 2 - [Title] (Priority: P2)
[Same structure as Phase 3]

---

## Phase N: Polish & Cross-Cutting Concerns

- [ ] TXXX [P] Documentation updates
- [ ] TXXX Code cleanup and refactoring
- [ ] TXXX Performance optimization
- [ ] TXXX Security hardening

---

## Dependencies & Execution Order

### Phase Dependencies
- Setup (Phase 1): No dependencies
- Foundational (Phase 2): Depends on Setup - BLOCKS all user stories
- User Stories (Phase 3+): All depend on Foundational phase
- Polish (Final Phase): Depends on all desired user stories

### Parallel Opportunities
[List which tasks can run in parallel]

## Implementation Strategy

### MVP First (User Story 1 Only)
1. Complete Setup + Foundational
2. Complete User Story 1
3. STOP and VALIDATE
4. Deploy/demo if ready

### Incremental Delivery
1. Setup + Foundational -> Foundation ready
2. User Story 1 -> MVP
3. User Story 2 -> Increment
4. Each story adds value without breaking previous stories
```

---

## Phase 4: Save Tasks Comment

**Step 1: Post or Update Comment**
1. Check if the "tasks comment" row in the issue body already has a reference:
   - If yes: **edit** that existing comment with the new tasks content
   - If no: **post** a new comment on the issue with the tasks content, then update the issue body table "tasks comment" row with the comment URL
2. If save failed: TERMINATE with `Err:TASKS_SAVE_FAILED`

---

## Phase 5: Completion

**Step 1: Set Label**
- Set label `speckit:tasks_ready` on the issue
- Remove any `Err:*` labels if workflow completed successfully

**Step 2: Report**
Post a summary comment on the issue:

```markdown
## Speckit: Task Breakdown Complete

**Tasks Comment:** (link)
**Total Tasks:** [count]

**Tasks per Phase:**
| Phase | Tasks | Parallel |
|-------|-------|----------|
| Setup | [n] | [n] |
| Foundational | [n] | [n] |
| US1 - [title] | [n] | [n] |
| US2 - [title] | [n] | [n] |
| Polish | [n] | [n] |

**MVP Scope:** Phase 1-3 (Setup + Foundational + User Story 1)
**Format Validation:** All tasks follow checklist format (checkbox, ID, labels, file paths)

**Next Steps:**
- Run `/speckit.implement` to begin implementation
```

---

## Task Generation Rules

**Tests are OPTIONAL**: Only generate test tasks if explicitly requested in the spec or user requests TDD approach.

**Logic vs UI Task Separation (CRITICAL):**
- A single task must NEVER mix logic and UI work. If a feature requires both, split it into separate tasks.
- **Logic tasks** (no [UI] marker): data models, services, business logic, API endpoints, middleware, controllers, hooks, utilities, backend configuration.
- **UI tasks** ([UI] marker): templates, components, views, pages, layouts, CSS/styles, animations, transitions, design system integration, accessibility markup.
- UI tasks should be listed AFTER their corresponding logic tasks within each phase — the UI agent needs the logic layer to exist before wiring to it.
- If a task seems to require both (e.g., "Create user profile page with data fetching"), split it:
  - Logic task: "Implement UserProfile service/hook with data fetching in src/services/userProfile.ts"
  - UI task: "Create UserProfile page component wired to UserProfile service in src/pages/UserProfile.tsx"

**Within Each User Story:**
- Tests (if included) MUST be written and FAIL before implementation
- Models before services
- Services before endpoints
- Core logic implementation before UI implementation
- **All logic tasks before UI tasks** (UI wires to the logic layer)
- Story complete before moving to next priority

**Notes:**
- [P] tasks = different files, no dependencies
- [UI] tasks = handled by the UI implementation agent, not the logic agent
- [Story] label maps task to specific user story for traceability
- Each user story should be independently completable and testable
- Commit after each task or logical group
- Stop at any checkpoint to validate story independently
- Avoid: vague tasks, same file conflicts, cross-story dependencies that break independence, mixing logic and UI in a single task
- The tasks comment should be immediately executable - each task must be specific enough that an LLM can complete it without additional context
