# Hermes-Based TheraSkill Practitioner Assistant Plan

Updated: 2026-06-28

## Positioning

TheraSkill should be positioned first as a practitioner-facing decision-support layer, not as an autonomous patient-facing therapist. The first product form is a Hermes-based clinical education and consultation copilot that retrieves, explains, and audits therapeutic skill options for a case description or consultation transcript.

The core claim is:

> TheraSkill externalizes therapeutic process knowledge into an auditable skill layer that a Hermes-based practitioner copilot can use to select, explain, simulate, and evaluate therapeutic actions, while keeping final clinical responsibility with trained professionals.

This framing supports three applications with one shared action layer:

1. Practitioner assistant: retrieve and explain relevant therapeutic skill bundles for a patient scenario.
2. Simulation: generate patient or practitioner-side role-play grounded in scenario states and selected skills.
3. Trainee education: evaluate trainee-patient dialogue records against expected key points, safety gates, and skill execution rules.

## Current Inputs

Project root:

```text
/vepfs-mlp2/c20250513/241404044/users/roytian/TheraSkillAgent
```

Trusted Chinese source:

```text
data/skill_library/skills.jsonl
data/skill_library/skills_retrieval_index.jsonl
data/skill_library/tree_nodes.jsonl
data/skill_library/manifest.json
data/benchmark/client_view/items.jsonl
data/benchmark/case_view/items.jsonl
data/benchmark/client_view/answer_key.jsonl
data/benchmark/case_view/answer_key.jsonl
```

Verified current counts from `data/skill_library/manifest.json` and `data/benchmark/manifest.json`:

- 52,070 full skill cards.
- 52,070 retrieval-index rows.
- 2,245 skill-type tree nodes.
- 9 skill categories.
- 500 benchmark scenarios.
- 500 scenario items in each legacy benchmark view.
- 500 answer-key rows per view.

Hermes target checkout:

```text
/vepfs-mlp2/c20250513/241404044/users/roytian/hermes-agent-main
```

Hermes integration constraints observed from `AGENTS.md` and `CONTRIBUTING.md`:

- Keep Hermes core narrow.
- Prefer edge integrations over core-loop changes.
- New model tools should be service-gated or implemented as plugin/MCP when possible.
- If a built-in tool is added in the local fork, it must self-register in `tools/*.py` and be exposed through `toolsets.py`.

## Recommended Architecture

Use a two-layer architecture:

```text
Hermes Agent
  -> TheraSkill tool surface
     -> TheraSkill service/backend
        -> skill library loader
        -> lexical/dense/vector indexes
        -> query understanding
        -> safety/risk gate
        -> range-first retrieval
        -> bundle selector
        -> explanation generator
        -> simulation/evaluation modules
```

The TheraSkill domain backend should live in `TheraSkillAgent`, while Hermes should call it through a small stable interface. This keeps therapeutic logic, experiments, and benchmark code reproducible outside Hermes and avoids coupling the research code to Hermes internals.

Recommended implementation order:

1. Build `theraskill_agent/` backend inside `TheraSkillAgent`.
2. Expose a local HTTP or MCP server from `TheraSkillAgent`.
3. Add a thin Hermes-facing wrapper only after the backend API stabilizes.
4. Use Hermes skills/prompts to define the practitioner-facing behavior and safety boundary.

## Tool Surface

Minimum Hermes-facing tools for the first practitioner assistant:

### `theraskill_recommend_bundle`

Input:

```json
{
  "query": "case description or consultation question",
  "query_perspective": "client_first_person | case_third_person | transcript",
  "top_k": 10,
  "bundle_size": 3,
  "language": "zh"
}
```

Output:

```json
{
  "risk_flags": {},
  "case_understanding": {},
  "selected_skills": [],
  "supporting_skills": [],
  "fit_reasons": [],
  "safety_notes": [],
  "next_questions": []
}
```

### `theraskill_get_skill`

Returns the full skill card, including state, activation, execution, safety, trace, and tree path.

### `theraskill_explain_plan`

Converts retrieved skills into practitioner-facing guidance:

- why this bundle fits;
- what to assess first;
- what to try;
- what not to do;
- when to escalate;
- how confident the system is.

Second-stage tools:

- `theraskill_evaluate_transcript`: evaluate trainee-patient dialogue records.
- `theraskill_simulate_patient`: generate role-play patient turns from scenario state.
- `theraskill_run_benchmark`: run fixed benchmark profiles for reproducible evaluation.

Do not expose 52,070 skills directly as Hermes skills. They are data records, not procedural Hermes skills. Hermes should access them through retrieval tools.

## Retrieval Design

Use the range-first retrieval plan already documented in `docs/query_understanding_retrieval_redesign.md`.

Pipeline:

```text
query
  -> query understanding
  -> safety/risk gate
  -> tag/category/tree range planning
  -> parallel candidate generation
  -> candidate fusion
  -> LLM or rules-assisted bundle selection
  -> practitioner-facing explanation
```

The production default should not be tree-only. Tree routing is useful for explainability and search-space control, but wrong routing can kill recall. Keep a global fallback channel.

Suggested first retrieval channels:

- tag channel: top 120-200;
- tree channel: top 120-200;
- category channel: top 80-150;
- global hybrid fallback: top 50-100;
- final rerank: 30-50 candidates.

## Practitioner Assistant Output Contract

The assistant should not answer as if it is treating the patient directly. It should speak to the physician, counselor, or supervisor.

Recommended output sections:

1. Case understanding: concise formulation of presenting problem, context, goals, and uncertainty.
2. Risk and safety check: high-risk cues, missing information, escalation triggers.
3. Recommended therapeutic focus: primary and supporting directions.
4. Why the evidence fits: grounded in `activation_rule`, `state_rule`, and benchmark-style key points.
5. How to use it: steps from `execution_rule.intervention_steps`.
6. Avoid: contraindications and `must_not_do`.
7. Suggested next clinician questions: assessment or clarification questions before intervention.

Internal logs may retain reproducibility handles for debugging. User-facing physician notes, model-facing evaluation rubrics, and LLM-as-judge inputs should contain only the scenario, guidance, rubric, and redacted evidence text needed for evaluation.

Boundary wording:

- Use "may consider", "is consistent with", "suggests", and "requires clinician judgment".
- Avoid "diagnose", "treat", "guarantee", "the correct intervention is".
- For crisis content, prioritize risk assessment, local emergency resources, and supervisor escalation.

## Scenario 1 Evaluation: Practitioner-Facing Clinical Assistant

Treat both retrieval and generation as layers of the same practitioner-assistant scenario:

```text
case scenario
  -> retrieve skill-card evidence
  -> generate physician-facing guidance note
```

### Layer A: evidence utility

Object evaluated: retrieved skill-card evidence before final generation.

Main metrics:

- Key-point coverage@K: whether the retrieved cards cover the hidden answer-key key points.
- Safety evidence coverage@K: whether the retrieved cards contain required risk assessment, escalation, contraindication, or crisis-handling evidence when the scenario requires it.

Report only rubric-level evidence coverage for this layer: key-point coverage and safety evidence coverage. Internal construction metadata and non-release artifacts stay outside the reported metrics.

### Layer B: physician-guidance quality

Object evaluated: final physician-facing note.

Primary evaluation method:

- LLM-as-judge for scalable scoring.
- Expert review for judge calibration, high-risk cases, and final claims.

Judge-visible inputs:

- patient or case scenario;
- generated physician-facing note;
- rubric derived from hidden key points, safety requirements, and must-avoid constraints;
- optionally, redacted retrieved skill-card content for grounding checks.

Judge input constraints:

- no construction metadata;
- no provenance traces;
- redacted evidence text only when grounding checks require retrieved content.

Main dimensions:

- Clinical quality: fit to case and coherent practitioner-facing formulation.
- Safety correctness: correct handling of risk, contraindications, escalation, and professional boundaries.
- Actionability and educational usefulness: concrete next questions, usable intervention directions, and brief rationale.
- Overreach flag: unsupported diagnosis, medication advice, treatment guarantees, or unsupported clinical claims.

GT reference skills may be used internally to construct and audit key points, but model-facing and judge-facing inputs should only contain the scenario, rubric, generated output, and redacted evidence text.

## Scenario 2 Evaluation: Training Simulation

The second scenario is a training simulator. It is not evaluated as a retrieval task and should not reuse Scenario 1's evidence-coverage metrics. The evaluated object is the behavior of the simulator during a role-play session.

Primary scope: patient simulation for trainee practice. Practitioner simulation can be reported as an optional exemplar-generation subtrack, but it should not be treated as a clinical gold standard.

```text
scenario + hidden patient state + rubric
  -> trainee or scripted clinician turns
  -> simulated patient turns
  -> session-level evaluation
```

### Inputs

Use the existing scenario set as the first source of cases. For each simulation item, prepare:

- scenario summary;
- patient state, affect, context, and goals;
- expected key points that should be revealed when appropriately elicited;
- information that should not be volunteered immediately;
- risk state and escalation triggers;
- persona constraints and communication style.

### Main Metrics

- Case fidelity: whether the simulated patient remains consistent with the assigned scenario, patient state, goals, and affect.
- Progressive disclosure quality: whether key information is revealed when clinically appropriate questions are asked, without dumping all hidden key points at the start.
- Interaction realism: whether turns sound like a plausible patient rather than a clinician, supervisor, or answer-key narrator.
- Safety-trigger behavior: whether risk-related cues are revealed, maintained, and escalated consistently when the scripted or trainee interaction reaches safety-relevant content.
- Controllability: whether the simulator follows the same case constraints across repeated runs and different trainee questioning styles.

### Evaluation Protocol

Use two complementary test sets:

- Scripted probe sessions: fixed clinician-turn scripts covering rapport, clarification, direct risk questions, poor questions, and premature advice. These make simulator behavior comparable across systems.
- Free role-play sessions: human or model trainee interactions used to test realism, robustness, and session flow.

Primary scoring should use LLM-as-judge at the session level, with expert review on a stratified subset and all high-risk cases. The judge sees the scenario, hidden rubric, clinician turns, and simulator turns. Scores should be reported per metric above, plus a safety-failure rate for high-risk items.

### Practitioner Exemplar Subtrack

If we generate practitioner-side role-play examples, evaluate them as educational exemplars:

- alignment with scenario key points and safety requirements;
- clarity of therapeutic rationale;
- appropriate professional boundaries;
- usefulness as a teaching example.

These examples should be labeled as training references, not ground truth clinical responses.

## Scenario 3 Evaluation: Trainee Transcript Feedback

The third scenario evaluates whether TheraSkill can review trainee-patient communication records and produce educational feedback. This is the closest fit for the brain hospital intern data, but real-data evaluation requires de-identification, access control, and expert annotation.

```text
de-identified trainee-patient transcript
  -> retrieve rubric-relevant skill evidence
  -> generate trainee feedback report
  -> compare against expert educational review
```

### Inputs

Each evaluation item should contain:

- de-identified trainee-patient transcript;
- optional case description or referral context;
- expert rubric containing expected key points, safety requirements, missed opportunities, and must-avoid behaviors;
- expert-written or expert-approved feedback notes for a subset used in calibration.

### Output

The system output should be an educational feedback report:

1. brief case and interaction summary;
2. trainee strengths grounded in transcript evidence;
3. missed key points or insufficient exploration;
4. safety and boundary concerns;
5. concrete alternative phrasing;
6. prioritized learning goals.

### Main Metrics

- Issue detection: precision, recall, and F1 against expert-labeled feedback points, evaluated at the level of rubric items rather than internal library fields.
- Safety-gap detection: sensitivity to missed risk assessment, escalation needs, contraindications, and unsafe or over-directive statements. Severe misses should be reported separately as gating failures.
- Feedback grounding: whether each critique or praise point is supported by transcript evidence.
- Educational usefulness: whether the report gives specific, prioritized, non-punitive feedback and concrete alternative wording.
- Overreach control: whether the system avoids unsupported diagnosis, moral judgment, medication advice, or claims not grounded in the transcript.

### Evaluation Protocol

Use a two-stage benchmark:

- Synthetic-stress subset: generated or templated trainee transcripts from the 500 scenarios, with controlled omissions such as missed risk assessment, premature advice, weak validation, or failure to clarify goals. This subset supports repeatable testing because expected issues are known.
- Expert-annotated real subset: de-identified intern transcripts reviewed by experienced clinicians or supervisors. This subset supports external validity and should be used for final claims.

LLM-as-judge can score feedback grounding, educational usefulness, and overreach at scale. Expert review is required for safety-gap labels, final calibration, and any claim involving real trainee data. Report agreement between LLM judge and experts before relying on judge-only scores.

## Hermes Integration Options

### Option A: Local research fork tool

Add `tools/theraskill_tool.py` in `hermes-agent-main` and expose it in a dedicated `theraskill` toolset. The tool calls the TheraSkill backend over localhost.

Use when:

- fast local demo matters more than upstream compatibility;
- we control the deployment environment;
- the TheraSkill backend is already running.

Risk:

- adds model-tool surface to Hermes;
- must keep tool schema compact;
- requires `toolsets.py` changes.

### Option B: MCP server

Expose TheraSkill as an MCP server and configure Hermes to connect through its MCP client.

Use when:

- we want a clean boundary;
- we may reuse TheraSkill from non-Hermes hosts;
- we want to avoid patching Hermes core.

This is the recommended longer-term route.

### Option C: Hermes skill + CLI command

Provide a Hermes skill that instructs the agent to call a `theraskill` CLI command through terminal/file tools.

Use when:

- only a prototype is needed;
- structured tool calling is not yet required.

Risk:

- weaker structured I/O;
- harder to evaluate automatically.

## MVP Milestones

### M0: Backend contract

Deliver:

- `theraskill_agent/` package skeleton.
- loader for `data/skill_library`.
- `recommend_bundle(query)` returning internal skill references plus redacted, physician-facing evidence summaries.
- reproducible config under `configs/`.

Validation:

- run on 20 practitioner-assistant benchmark scenarios;
- verify all internal skill references resolve to current skill cards;
- log latency, retrieval profile, and redaction status.

### M1: Practitioner assistant demo

Deliver:

- Hermes-accessible `recommend_bundle` and `get_skill`.
- practitioner-facing response template.
- audit log per query.

Validation:

- run on 50 benchmark items;
- report key-point coverage of retrieved evidence, safety evidence coverage, and qualitative safety notes.

### M2: Full benchmark run

Deliver:

- BM25, dense, hybrid, and metadata-aware profiles.
- results for the practitioner-assistant scenario.
- summary JSON and paper-ready table.

Validation:

- use fixed current Chinese library checksums;
- report evidence key-point coverage and safety evidence coverage as retrieval-layer metrics;
- report final physician-guidance quality with LLM-as-judge and expert calibration.

### M3: Trainee evaluation prototype

Deliver:

- transcript input schema;
- move detector;
- rubric output;
- de-identified sample reports.

Validation:

- expert review on a small subset;
- record disagreements and revision points.

### M4: Simulation prototype

Deliver:

- patient simulator for benchmark scenarios;
- role-play session logs;
- trainee evaluation loop.

Validation:

- check persona consistency, key-point reveal policy, and safety-trigger behavior.

## Paper Wording Changes

Use:

- practitioner-facing copilot;
- clinical education and supervision support;
- auditable therapeutic action layer;
- skill selection;
- skill bundle recommendation;
- safety-gated action planning;
- Hermes-based runtime;
- decision support, not autonomous treatment.

Avoid or qualify:

- autonomous therapist;
- treatment recommendation;
- diagnosis;
- clinical efficacy;
- real-world safety;
- patient-facing deployment;
- ground truth without expert-review qualification.

## Immediate Next Steps

1. Keep the paper framed around `Scenario 1: Practitioner-Facing Clinical Assistant`.
2. Build the benchmark rubric around key points, safety requirements, and must-avoid constraints, with hidden reference skills used only for construction and audit.
3. Implement the backend before touching Hermes core.
4. Expose redacted skill-card evidence to the generation layer and keep reproducibility handles only in logs.
5. Calibrate LLM-as-judge with expert review, especially for high-risk scenarios.
6. Add trainee evaluation and simulation as extension applications, not current evidence.
