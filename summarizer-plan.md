# Clinical Summary Improvement: Review and Release Plan

## Purpose

This document explains the preparation completed for the clinical-summary improvement work, how the development collection is used, and the approval process before any production change.

## Current position

The current production summaries remain unchanged. A V3 improvement package has been prepared for technical and clinical review. It is not yet approved for production deployment.

## Work prepared by Chris

Chris has prepared a revised clinical-summary approach to make AI output more traceable, consistent, and clinically reviewable.

| Area | What has been prepared | Expected benefit |
|---|---|---|
| Clinical scoring | A central set of scoring guardrails for Safety, Severity, Urgency, Criticality, and risk bands. | Applies one agreed scoring approach across summaries. |
| VALD interpretation | Uses the average of valid trials within a session and recalculates left/right asymmetry from raw values. | Reduces the risk of misleading results from selecting a single trial or relying on an incorrect derived value. |
| Missing measurements | Treats zero, blank, null, or missing left/right readings as a testing gap rather than a patient deficit. | Prevents false 100% asymmetry concerns. |
| Areas of Concern | Separates documented patient findings from missing data, testing gaps, and programme-improvement observations. | Keeps risk scores focused on patient evidence. |
| Evidence language | Requires the report to distinguish objective evidence from clinical inference. | Makes it clearer what was measured, what was documented, and what is an interpretation. |
| Summary fidelity | Adds checks between the detailed assessment and the concise dashboard summary. | Helps detect dropped concerns or changed scores/numbers during summarisation. |
| Development workflow | Supports isolated development testing through the orchestrator and dev output collections. | Allows review without overwriting production summaries. |

## What is `new-summary-dev`?

`new-summary-dev` is an isolated MongoDB collection for development and validation output.

- Production dashboard summaries use the production `new-summary` collection.
- Development test summaries are written to `new-summary-dev`.
- It allows the team and clinical reviewers to compare current and proposed outputs safely.
- A document in `new-summary-dev` is a test result only. It is not a clinically approved or production result.

## What we need to agree with the clinical team

Before changing prompts, scoring rules, or production summaries, the clinical team should approve the following:

1. What qualifies as an Area of Concern.
2. How Safety, Severity, and Urgency should be scored.
3. When VALD asymmetry is clinically meaningful.
4. How to handle single-session VALD results and missing-side measurements.
5. How clinical notes, imaging, functional tests, symptoms, and VALD measurements should be weighted.
6. Which findings should be presented as objective evidence and which as clinical inference.
7. What a correct summary should look like for representative patient scenarios.

## Proposed delivery plan

| Phase | Activity | Output | Approval gate |
|---|---|---|---|
| 1. Baseline audit | Review a representative sample of current summaries against source clinical notes and VALD records. | List of confirmed output issues and examples. | Clinical team confirms priorities. |
| 2. Clinical rules workshop | Review the proposed scoring guardrails and evidence rules with doctors. | Agreed clinical scoring and evidence policy. | Doctors approve the rules. |
| 3. Acceptance examples | Define examples of correct and incorrect summary output for common cases. | Test cases with expected outcomes. | Doctors approve expected output. |
| 4. Technical completion | Resolve identified rule conflicts, secure deployment artifacts, and complete automated tests. | Review-ready implementation. | Technical review confirms readiness for dev validation. |
| 5. Development validation | Generate approved test cases into `new-summary-dev` and compare old and new output. | Side-by-side clinical comparison. | Doctors approve output quality. |
| 6. Controlled production release | Deploy approved changes and regenerate summaries in monitored batches. | Production rollout report and monitoring results. | Final release approval. |

## Important safeguards

- No production summary will be overwritten during the review stage.
- Clinical scoring rules will not be changed based only on engineering judgement.
- Each proposed change will be tested in `new-summary-dev` before production use.
- Doctors will review representative output before a production regeneration begins.
- Production regeneration will be performed in controlled batches after approval.

## Recommended client decision

Please approve Phase 1 and Phase 2: a baseline output audit and a clinical rules workshop. After the clinical team agrees the scoring and evidence rules, the technical team will implement only the approved changes and demonstrate the results in `new-summary-dev` before requesting production release approval.
