# V3 Clinical Summary: Full Context and Change Summary

## What this comparison covers

- **Old version:** `V2/eba_agent_instance.py`, copied from the EC2 production environment.
- **Proposed V3 version:** `LLM/report-gen/eba_agent.py`, prepared locally for review.
- **Purpose:** reduce unsupported Areas of Concern, incorrect scores, false VALD asymmetry, incomplete patient context, and loss of information between the detailed report and dashboard summary.

V3 is a proposed improvement package. It is not yet approved for production use.

## The old workflow

The old workflow used one very large AI prompt to read clinical notes and VALD data, generate a detailed report, then create a shorter dashboard summary. Most clinical rules were written directly inside the prompt. Some rules were repeated in more than one place, and some values supplied to the AI could be incomplete, unsorted, or based on derived VALD fields.

This created several risks:

- the AI could apply different versions of the same scoring rule;
- a score could be mathematically inconsistent;
- the strongest VALD trial could be selected instead of a representative session result;
- a missing left/right measurement could look like a major asymmetry;
- a single measurement could be described as a trend;
- an old clinical or imaging finding could be presented as current;
- missing tests or programme observations could become patient risks;
- the concise dashboard summary could drop or alter information from the detailed report.

## V3 design approach

V3 changes the system from a broad, AI-led prompt toward a controlled workflow:

1. Clinical rules are collected in one guardrail file.
2. The prompt receives the guardrails and the patient timeline.
3. VALD values are prepared before they reach the AI.
4. The detailed report is checked for scoring and evidence issues.
5. The concise dashboard report is compared with the detailed report.
6. A result with validation problems can be blocked from publication.

The important principle is: **the prompt tells the AI how to reason; backend checks should verify what is safe to save.**

## Changes to scoring rules

| Old behavior | V3 change | Why it matters |
|---|---|---|
| Formula, score bands, and rules appeared in multiple prompt sections. | Rules are stored in `LLM/report-gen/clinical_guardrails.json` and injected into the prompt. | Reduces conflicting instructions and gives reviewers one place to approve or update rules. |
| The criticality example contained an incorrect rounded result. | The example is corrected: Safety 6, Severity 7, Urgency 6 produces 6.4. | Prevents the AI from learning an incorrect arithmetic example. |
| Overall risk could be based on an average that diluted a high-risk concern. | Numeric score can remain an average, but risk label is based on the highest individual concern. | A high-risk concern is not hidden by several low-risk concerns. |
| Safety, Severity, and Urgency definitions were mixed into a large prompt. | Definitions and guardrails are centralized in the guardrail file. | Doctors can review each scoring rule clearly. |
| Urgency 10 could be assigned from a named condition or alarming wording. | Urgency 10 requires documented time-critical findings, risk of irreversible harm, emergency escalation, and explicit rationale. | Reduces false emergency-level scores. |
| A trend could directly set a criticality range. | A trend informs Safety, Severity, or Urgency; the final score still uses the formula. | Prevents trajectory wording from bypassing score rules. |

### Proposed formula

```text
Criticality = (Safety × 2.2 + Severity × 2.0 + Urgency × 1.3) ÷ 5.5
```

This formula and its weights remain proposed until doctors approve them.

## Changes to patient context and timeline

| Old behavior | V3 change | Why it matters |
|---|---|---|
| The prompt could receive only two clinical records from an unsorted list but describe them as recent. | V3 sends a full chronological record set, with clear notice if records are omitted due to size. | The AI has the context needed to describe baseline, latest status, and treatment journey accurately. |
| Date parsing failures could appear as a zero-week duration. | V3 labels unparseable dates as unknown and instructs the AI not to infer a phase from them. | Avoids falsely treating a patient as being at the start of rehabilitation. |
| Care gaps used only clinical-visit dates. | V3 includes VALD session dates as patient contact. | Avoids claiming a care gap when the patient attended an assessment. |
| A simple number extraction could be labelled as current pain. | V3 marks automatic extraction as unvalidated and requires the AI to verify it against the actual note. | Prevents sleep, goal, or function scores being presented as pain scores. |

## Changes to VALD data handling

| Old behavior | V3 change | Why it matters |
|---|---|---|
| Highest or best trial could be used for a session. | Use the mean of valid trials for each side when available. | Averages are more representative and reduce cherry-picking. |
| Pre-calculated asymmetry could be accepted without checking raw values. | Recalculate asymmetry using `abs(left - right) / max(left, right) × 100`. | Makes the percentage traceable and catches incorrect derived values. |
| The output did not clearly say how asymmetry was calculated. | Include calculation basis, left value, right value, and session date when using VALD evidence. | Clinicians can check the evidence. |
| Missing, null, blank, or zero measurement on one side could create a 100% asymmetry. | Treat this as one-sided or incomplete data; do not calculate or score asymmetry. | Prevents a false and alarming patient deficit. |
| One VALD session could be described as regression, plateau, or persistent deficit. | One session may describe that measurement only; trend claims require at least two comparable sessions. | Prevents unsupported trajectory claims. |
| Very large asymmetry could be scored from the number alone. | V3 requires bilateral-value confirmation, recalculation, record sense-check, and corroborating evidence before scoring. | Reduces false high-risk concerns caused by measurement artifacts. |

## Changes to evidence and clinical wording

| Old behavior | V3 change | Why it matters |
|---|---|---|
| The report could state a score without explaining whether the evidence was measured or inferred. | Use `Objective (source and date):` for measured/documented facts and `Clinical inference (reason):` for interpretation. | Makes evidence type visible to the clinician. |
| Internal labels such as `[CONFIRMED]` and `[UNVERIFIED/UNASSESSED]` could appear in clinician output. | Replace them with plain clinical language. | Reports are easier to read and do not look like internal system output. |
| VALD could be treated as the default strongest evidence source. | V3 uses an evidence-weighting framework: clinical examination, imaging, function, symptoms, and VALD are evaluated by quality and corroboration. | Prevents valid clinical evidence from being ignored because it has no VALD number. |
| Structural findings mentioned in a note could be treated as current. | A structural or imaging finding requires a recoverable performed date before it affects score. | Prevents old or undated investigations being presented as current risk. |
| A single data point could be described as unreliable simply because it was one session. | V3 treats a valid measurement as objective evidence but limits trend claims. | Keeps valid data while being honest about its limits. |

## Changes to Areas of Concern

| Old behavior | V3 change | Why it matters |
|---|---|---|
| Missing VALD tests or unassessed areas could become scored concerns. | Missing data belongs in Critical Information Needed, not Areas of Concern. | Absence of evidence is not evidence of a patient deficit. |
| Under-addressed muscles could become concerns based only on a condition template. | A muscle must have a patient-specific sign or measurement and lack of protocol coverage before it is highlighted. | Reduces boilerplate and invented risk. |
| A fixed quota could encourage the AI to create several concerns. | There is no minimum number of concerns or under-addressed muscles. | The report can correctly say there are no supported concerns. |
| ROM and force findings in one region could become duplicate concerns. | Combine related findings when they describe the same clinical issue. | Produces a clearer, less inflated risk profile. |
| Programme critiques, adherence gaps, or testing outside a phase could be scored as patient risk. | Put programme and information issues in their own sections unless there is a documented patient finding. | Keeps patient risk separate from service or documentation improvement. |
| An unrelated secondary issue could be handled inconsistently. | V3 proposes a relevance gate based on effect on the primary rehabilitation. | Prevents unrelated symptoms from distorting the main rehabilitation risk summary. |
| A gap in care needed VALD decline to be included. | V3 allows objective or documented clinical deterioration, not only VALD decline. | Captures clinically meaningful change from notes, ROM, function, or measurement. |

## Changes to citations and research evidence

| Old behavior | V3 change | Why it matters |
|---|---|---|
| The AI was asked to generate a standalone references section and URLs. | Use Google Search grounding metadata instead of model-generated reference lists. | Reduces fabricated references and URLs. |
| Literature citations could be attached to patient-specific measurements. | Patient values must point to patient records; literature supports only general clinical claims. | Prevents misleading external validation of a patient's own result. |
| Future-dated or example citations could appear in output. | Do not use future publications or reuse illustrative examples without real search support. | Reduces citation hallucinations. |

## Changes to validation and publication

| Check | Proposed location | Intended purpose |
|---|---|---|
| Formula, score relationship, risk-band, evidence, and output-format checks | `LLM/report-gen/scoring_utils.py` and `LLM/report-gen/eba_validator.py` | Find invalid detailed-report concerns before publication. |
| Detailed-to-concise numeric and concern comparison | `LLM/report-gen/concise_validator.py` | Detect dropped concerns or changed scores/measurements in dashboard output. |
| Publication decision | `LLM/report-gen/run_audit_by_stance_id.py` and `LLM/report-gen/push_report_to_mongo.py` | Block unsafe or incomplete output from being stored in MongoDB. |
| Job status | `api/summary_api.py` and `api/job_store.py` | Ensure a blocked or failed output is visible as failed/review-required, not completed. |
| Safe test environment | `docker-compose.dev.yml` and `new-summary-dev` | Test output without changing production summary records. |

## What remains to be completed before production use

The V3 material provides a strong starting point, but it needs review and completion before it is merged or deployed:

1. Doctors must approve the formula, weights, score limits, evidence rules, and examples.
2. Conflicting rules must be removed so the prompt and guardrail file give the same instruction.
3. Every validation failure must produce a failed or review-required job status and prevent a MongoDB write.
4. Credentials and generated patient reports must be removed from source control and Docker build context.
5. Automated tests must run in a complete test environment.
6. Representative patient cases must be generated into `new-summary-dev` and clinically reviewed.
7. Production rollout must begin with a small monitored batch after clinical approval.

## Expected result after approved implementation

The expected output is not simply a different AI summary. It is a more auditable clinical summary where every scored concern has a clear reason:

- what the concern is;
- which patient note, test, or measurement supports it;
- the source date and body side where applicable;
- Safety, Severity, Urgency, and formula-checked Criticality;
- no false asymmetry from missing data;
- no trend claim from one session;
- no risk score based only on a missing test or programme critique;
- the same approved concern and numbers in the detailed report and dashboard summary.
