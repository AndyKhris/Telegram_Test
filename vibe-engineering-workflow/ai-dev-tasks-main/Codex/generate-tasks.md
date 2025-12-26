---
description: Generate a step-by-step task list from a PRD and save it under /tasks
argument-hint: PRD=<path>
---

# Rule: Generating a Task List from a PRD

## Goal

To guide an AI assistant in creating a detailed, step-by-step task list in Markdown format based on an existing Product Requirements Document (PRD). The task list should guide a developer through implementation.

## Output

- **Format:** Markdown (`.md`)
- **Location:** `/tasks/`
- **Filename:** `tasks-[prd-file-name].md` (e.g., `tasks-0001-prd-user-profile-editing.md`)

## Process

0. **Pre-flight Setup (run once per PRD; idempotent):** Verify the development environment is ready for this PRD. Ensure the language/toolchain and dependencies for the selected stack are installed, `.env.example` exists with placeholders, `.env` is gitignored, ask the user to add required keys in `.env`, and programmatically verify required keys are present before proceeding to Parent Task 1.

1. **Receive PRD Reference:**  
   The user points the AI to a specific PRD file.

   When invoked as a slash command, treat `$PRD` (if provided) as the path to the PRD file to use (e.g., `@/tasks/0003-prd-generate-images.md`).

2. **Analyze PRD:**  
   The AI reads and analyzes the functional requirements, user stories, and other sections of the specified PRD.

3. **Assess Current State:**  
   Review the existing codebase to understand existing infrastructure, architectural patterns and conventions. Also, identify any existing components or features that already exist and could be relevant to the PRD requirements. Then, identify existing related files, components, and utilities that can be leveraged or need modification.

4. **Phase 1: Generate Parent Tasks:**  
   Based on the PRD analysis and current state assessment, create the file and generate the main, high-level tasks required to implement the feature. Use your judgement on how many high-level tasks to use. It's likely to be about five tasks. Present these tasks to the user in the specified format (without sub-tasks yet). Inform the user: "I have generated the high-level tasks based on the PRD. Ready to generate the sub-tasks? Respond with 'Go' to proceed." And show the numbered list of parent tasks below for user to see.

5. **Wait for Confirmation:**  
   Pause and wait for the user to respond with "Go".

6. **Phase 2: Generate Sub-Tasks:**  
   Once the user confirms, break down each parent task into smaller, actionable sub-tasks necessary to complete the parent task. Ensure sub-tasks logically follow from the parent task, cover the implementation details implied by the PRD, and consider existing codebase patterns where relevant without being constrained by them.

7. **"Generate Unit Tests" as final Sub-Task for each Parent-task:**  
   Add "generate unit tests" as the final sub-task for each parent-task **except** the integration-test parent task (if present).

8. **Integration Tests for Vertical Features (final parent task):**  
   If the PRD represents a **vertical feature** (end-to-end user flow), add a **final parent task**
   named "Integration Tests" (or "End-to-End Integration Tests") with concrete sub-tasks.
   - These tests should exercise **real internal components together** (API + queue + worker),
     while **mocking external third-party APIs** unless a real test environment is explicitly available.
   - Add **"generate integration tests"** as the final sub-task for this parent task.
   - Place this parent task **last** in the task list.

9. **Release Smoke/E2E Suite (Ops slice PRD only):**  
   - If the PRD is specifically for the Ops feature "Release Smoke/E2E Suite" (post-deploy verification), ensure the task list includes a dedicated parent task (placed last) per the "Release Smoke/E2E Suite guidelines" below.
   - Create/update a runnable smoke/E2E entrypoint (e.g., `scripts/smoke.*`) that returns a non-zero exit code on failure.
   - Create/update `tests/smoke/` and/or `tests/e2e/` and add coverage for the PRD's critical end-to-end flows.
   - Document how to run locally and against staging (required env vars, safety notes, timeouts).
   - Ensure the suite is safe-by-default (no spamming public chats, no destructive actions) and does not log secrets.

10. **Identify Relevant Files:**  
   Based on the tasks and PRD, identify potential files that will need to be created or modified. List these under the `Relevant Files` section, including corresponding test files if applicable.

11. **Generate Final Output:**  
   Combine the parent tasks, sub-tasks (including generate tests), relevant files, and notes into the final Markdown structure.

12. **Save Task List:**  
    Save the generated document in the `/tasks/` directory with the filename `tasks-[prd-file-name].md`, where `[prd-file-name]` matches the base name of the input PRD file (e.g., if the input was `0001-prd-user-profile-editing.md`, the output is `tasks-0001-prd-user-profile-editing.md`).

## Output Format

The generated task list _must_ follow this structure:

```markdown
## Relevant Files

- `path/to/potential/file1.ts` - Brief description of why this file is relevant (e.g., Contains the main component for this feature).
- `path/to/file1.test.ts` - Unit tests for `file1.ts`.
- `path/to/another/file.tsx` - Brief description (e.g., API route handler for data submission).
- `path/to/another/file.test.tsx` - Unit tests for `another/file.tsx`.
- `lib/utils/helpers.ts` - Brief description (e.g., Utility functions needed for calculations).
- `lib/utils/helpers.test.ts` - Unit tests for `helpers.ts`.
- `tests/unit/` - Default location for unit tests if no repo-specific convention exists.
- `tests/integration/` - Default location for integration tests if no repo-specific convention exists.
- `tests/smoke/` - Default location for smoke tests if no repo-specific convention exists.
- `tests/e2e/` - Default location for E2E tests if no repo-specific convention exists.

### Notes

- Unit tests should typically be placed alongside the code files they are testing (e.g., `MyComponent.tsx` and `MyComponent.test.tsx` in the same directory).
- Use `npx jest [optional/path/to/test/file]` to run tests. Running without a path executes all tests found by the Jest configuration.
- For **vertical features**, include a final **Integration Tests** parent task (placed last).
- For the Ops PRD **Release Smoke/E2E Suite**, include a final **Release Smoke/E2E Suite** parent task (placed last).

## Instructions for Completing Tasks

**IMPORTANT:** As you complete each task, you must check it off in this markdown file by changing `- [ ]` to `- [x]`. This helps track progress and ensures you don't skip any steps.

Example:
- `- [ ] 1.1 Read file` → `- [x] 1.1 Read file` (after completing)

Update the file after completing each sub-task, not just after completing an entire parent task.

## Tasks

- [ ] 0.0 Pre-flight Setup (run once per PRD; idempotent)
  - [ ] 0.1 Ensure dependencies installed (stack-appropriate: e.g., `pip install -r requirements.txt`, `npm ci`, `pnpm i`)
  - [ ] 0.2 Ensure `.env.example` exists with placeholders; ensure `.env` is gitignored
  - [ ] 0.3 Ask user to add required keys and api keys in `.env` and confirm
  - [ ] 0.4 Programmatically verify required keys are present (fail fast if missing)

- [ ] 1.0 Parent Task Title
  - [ ] 1.1 [Sub-task description 1.1]
  - [ ] 1.2 [Sub-task description 1.2]
- [ ] 2.0 Parent Task Title
  - [ ] 2.1 [Sub-task description 2.1]
- [ ] 3.0 Parent Task Title (may not require sub-tasks if purely structural or configuration)
- [ ] N.0 Integration Tests (vertical feature only; last parent task)
  - [ ] N.1 [Integration test setup/execution]
  - [ ] N.2 generate integration tests
- [ ] N+1.0 Release Smoke/E2E Suite (Ops slice PRD only; last parent task)
  - [ ] N+1.1 Add runnable smoke entrypoint (e.g., `scripts/smoke.*`) with non-zero exit code on failure
  - [ ] N+1.2 Add/update smoke/E2E tests under `tests/smoke/` and/or `tests/e2e/`
  - [ ] N+1.3 Document local and staging run commands + safety notes
  - [ ] N+1.4 generate smoke/e2e tests

## Interaction Model

The process explicitly requires a pause after generating parent tasks to get user confirmation ("Go") before proceeding to generate the detailed sub-tasks. This ensures the high-level plan aligns with user expectations before diving into details.

## Target Audience

Assume the primary reader of the task list is a **junior developer** who will implement the feature with awareness of the existing codebase context.
