---
description: Generate a PRD under /tasks from a feature prompt or feature placeholder
argument-hint: FEATURE_PLACEHOLDER=<path> PROMPT="short description"
---

# Rule: Generating a Product Requirements Document (PRD)

## Goal

To guide an AI assistant in creating a detailed Product Requirements Document (PRD) in Markdown format, based on an initial user prompt. The PRD should be clear, actionable, and suitable for a junior developer to understand and implement the feature.

## Process

1.  **Receive Initial Prompt:** 

    The user either:
    - (a) provides a brief description or request for a new feature or functionality, **or**
    - (b) points you to a **feature placeholder file** created by `generate-features.md`, e.g. `@/tasks/features/0003-feature-generate-images.md`.

    When invoked as a slash command:
    - Treat `$FEATURE_PLACEHOLDER` (if provided) as the path to the feature placeholder file.
    - Treat `$PROMPT` (if provided) as the brief description / request for the feature.
    - If both are provided, use the feature placeholder as the primary input and use `$PROMPT` only to refine or clarify intent.

2. **Ask Clarifying Questions:**  
   Before writing the PRD, the AI *must* ask clarifying questions to gather sufficient detail. The goal is to understand the "what" and "why" of the feature, not necessarily the "how" (which the developer will figure out). Make sure to provide options in letter/number lists so I can respond easily with my selections.

   - If a **feature placeholder file** is provided, first read it fully and treat it as the starting point for the PRD. The placeholder will typically include sections like Problem, Outcome, Scope, Flow, Dependencies, Interfaces / Contracts (seeds), Acceptance Seeds, Non-Functional / Telemetry (seeds), and Risks & Open Questions.
   - In that case, focus clarifying questions on:
     - Filling in any missing or vague parts of those sections.
     - Confirming or adjusting the proposed scope, outcomes, and acceptance seeds.
     - Clarifying any risks or open questions.
   - Do **not** re-ask for information that is already clearly stated in the placeholder; instead, reference it and only probe where it is incomplete or ambiguous.

3. **Generate PRD:**  
   Based on the initial prompt, any feature placeholder file, and the user's answers to the clarifying questions, generate a PRD using the structure outlined below.

4. **Save PRD:**  
   Save the generated document as `[n]-prd-[feature-name].md` inside the `/tasks` directory.

   - If this PRD is being created from a feature placeholder file such as `/tasks/features/0003-feature-generate-images.md`, then:
     - Reuse the same `n` (`0003`) and the same `<feature-name>` slug (`generate-images`), producing:  
       `/tasks/0003-prd-generate-images.md`.
   - If there is no existing feature placeholder, pick the next available `n` (zero-padded 4-digit sequence starting from 0001, e.g., `0001-prd-user-authentication.md`, `0002-prd-dashboard.md`, etc.).

## Clarifying Questions (Examples)

The AI should adapt its questions based on the prompt and/or feature placeholder, but here are some common areas to explore:

- **Problem/Goal:** "What problem does this feature solve for the user?" or "What is the main goal we want to achieve with this feature?"
- **Target User:** "Who is the primary user of this feature?"
- **Core Functionality:** "Can you describe the key actions a user should be able to perform with this feature?"
- **User Stories:** "Could you provide a few user stories? (e.g., As a [type of user], I want to [perform an action] so that [benefit].)"
- **Acceptance Criteria:** "How will we know when this feature is successfully implemented? What are the key success criteria?"
- **Scope/Boundaries:** "Are there any specific things this feature *should not* do (non-goals)?"
- **Data Requirements:** "What kind of data does this feature need to display or manipulate?"
- **Design/UI:** "Are there any existing design mockups or UI guidelines to follow?" or "Can you describe the desired look and feel?"
- **Edge Cases:** "Are there any potential edge cases or error conditions we should consider?"
- **Non-Functional / Telemetry:** "Are there performance, security/privacy or logging/metrics requirements we should capture explicitly?"

When a feature placeholder already contains some of this information, reference those sections and only ask about what is missing or unclear.

## PRD Structure

The generated PRD should include the following sections:

1. **Introduction/Overview:** Briefly describe the feature and the problem it solves. State the goal.
2. **Goals:** List the specific, measurable objectives for this feature.
3. **User Stories:** Detail the user narratives describing feature usage and benefits.
4. **Functional Requirements:** List the specific functionalities the feature must have. Use clear, concise language (e.g., "The system must allow users to upload a profile picture."). Number these requirements.
5. **Non-Goals (Out of Scope):** Clearly state what this feature will *not* include to manage scope.
6. **Design Considerations (Optional):** Link to mockups, describe UI/UX requirements, or mention relevant components/styles if applicable.
7. **Technical Considerations (Optional):** Mention any known technical constraints, dependencies, or suggestions (e.g., "Should integrate with the existing Auth module"). Use this section to incorporate relevant Interfaces / Contracts seeds and Non-Functional / Telemetry seeds from the feature placeholder, if provided.
8. **Success Metrics:** How will the success of this feature be measured? (e.g., "Increase user engagement by 10%", "Reduce support tickets related to X").
9. **Open Questions:** List any remaining questions or areas needing further clarification. Include any unresolved items from the feature placeholder’s “Risks & Open Questions” section.

## Target Audience

Assume the primary reader of the PRD is a **junior developer**. Therefore, requirements should be explicit, unambiguous, and avoid jargon where possible. Provide enough detail for them to understand the feature's purpose and core logic.

## Output

*   **Format:** Markdown (`.md`)
*   **Location:** `/tasks/`
*   **Filename:** `[n]-prd-[feature-name].md` (use the same `n` as the feature placeholder)

## Final instructions

1. Do NOT start implementing the PRD.
2. Make sure to ask the user clarifying questions.
3. Take the user's answers to the clarifying questions (and any information from the feature placeholder, if provided) and improve the PRD.
