# Clinical Summary Improvement Plan

## Why we are doing this

The current summary can show Areas of Concern and scores that do not always match the patient’s latest notes, VALD results, or clinical context. Before changing production, we want the clinical team to agree what should count as a concern, what evidence is required, and how scores should be assigned.

The aim is simple: when a clinician opens a summary, they should be able to see what the concern is, where it came from, when it was recorded, and why it received that score.

## Work already prepared by Chris

Chris has prepared a V3 version of the summary workflow for review. It gives us a stronger starting point, but it is not yet approved for production.

| Area | What Chris has prepared |
|---|---|
| Scoring rules | One guardrail file for Safety, Severity, Urgency, Criticality, and risk bands. |
| VALD handling | Uses valid trial averages and recalculates left/right asymmetry from the recorded values. |
| Missing readings | Treats missing, blank, null, or zero readings as a data gap instead of a patient deficit. |
| Areas of Concern | Separates patient risks from missing tests, data gaps, and programme recommendations. |
| Evidence | Asks the report to state whether a finding is based on objective evidence or clinical interpretation. |
| Summary checks | Compares the detailed assessment with the short dashboard summary to find dropped concerns or changed numbers. |
| Dev setup | Uses an isolated development output collection for safe testing. |

## What `new-summary-dev` is for

`new-summary-dev` is a separate MongoDB collection used only for development testing.

- It does not replace the production `new-summary` collection.
- We can generate and review test summaries there without changing what clinicians see in production.
- It will be used to compare the current summary against the proposed summary for the same patient.
- Nothing from `new-summary-dev` moves to production until the clinical team approves it.

## Our proposed implementation plan

### Step 1: Agree the clinical rules before coding

We will hold a review with the clinical team and agree the rules below. These rules will become the accepted standard for the system.

| Topic | Decision needed from doctors |
|---|---|
| Areas of Concern | Which findings should be scored, and which should only be shown as an information gap or recommendation? |
| Scoring | What Safety, Severity, and Urgency levels should mean in common clinical situations? |
| VALD asymmetry | When is an asymmetry meaningful enough to be a concern, and when does it need supporting clinical evidence? |
| Single-session tests | What can be concluded from one session, and what requires a trend across multiple sessions? |
| Missing data | How should the report describe missing tests or incomplete measurements without treating them as a patient risk? |
| Evidence | What source, date, and wording should be required for a concern to be shown to clinicians? |
| Secondary symptoms | When should a new issue in another body area affect the main rehabilitation summary? |

**Output of Step 1:** a short signed-off clinical rules document and 10–15 example patient scenarios with expected results.

### Step 2: Build the approved rules into the backend

After the clinical rules are approved, we will implement the following:

1. Keep one approved scoring and evidence policy in a single backend configuration file.
2. Make the detailed AI report follow those rules for every Area of Concern.
3. Validate each concern before it can be saved:
   - score components must be valid;
   - the final score must match the agreed formula;
   - the concern must have supporting patient evidence and date;
   - missing data and programme observations must not become scored patient risks.
4. Validate VALD values before analysis:
   - use valid bilateral values only;
   - calculate asymmetry from the recorded left/right values;
   - use approved session/trial handling;
   - label incomplete testing as a gap, not a deficit.
5. Make the concise dashboard summary preserve approved concern headings, scores, evidence, and important measurements from the detailed report.
6. Make a failed validation return a failed job status and prevent any MongoDB summary write.
7. Remove credentials and generated patient reports from source control and use environment-based deployment configuration.

**Output of Step 2:** a reviewable backend change set with automated tests and no production data changes.

### Step 3: Test against real examples in development

We will choose a representative set of patients with different situations, for example:

- clear bilateral strength deficit;
- incomplete or missing VALD measurement;
- one VALD session only;
- improving and worsening symptoms;
- imaging or diagnosis mentioned in notes;
- secondary acute symptom;
- no clinically significant concern.

For every example, we will compare:

| Check | What we will verify |
|---|---|
| Source data | The summary uses the correct notes, date, VALD session, and side. |
| Concern inclusion | Only clinically approved concern types are scored. |
| Score | The score follows the agreed clinical rules and formula. |
| Evidence | The displayed wording clearly shows source and date. |
| Dashboard output | The short summary has the same concern, score, and important measurements as the detailed report. |

All test output will be saved only in `new-summary-dev`.

**Output of Step 3:** side-by-side old and new summaries for doctor review.

### Step 4: Clinical review and approval

Doctors will review the development summaries and confirm:

- the concerns are clinically appropriate;
- the score and risk level are appropriate;
- the evidence is clear and traceable;
- no important issue is missing;
- no unsupported issue is being added.

If the output is not acceptable, we update the agreed rules and repeat development validation. No production release happens at this stage.

### Step 5: Controlled production release

Once doctors approve the development results, we will:

1. deploy the approved backend version;
2. run a small controlled production batch first;
3. review the generated summaries and job results;
4. regenerate the remaining summaries in agreed batches;
5. monitor errors, validation blocks, and output quality;
6. retain the previous summaries until the batch is accepted.

## What we need from the client now

Please approve the following before implementation starts:

1. A clinical rules workshop with the doctors.
2. A representative set of patient cases for development comparison.
3. The proposed evidence and scoring review process.
4. Use of `new-summary-dev` for testing before production release.

Once these are agreed, we will provide a final technical implementation list and begin development.
