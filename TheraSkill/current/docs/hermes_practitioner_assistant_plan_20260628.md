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
- 500 client-view items.
- 500 case-view items.
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
3. Recommended skill bundle: primary and supporting skills with skill IDs.
4. Why these skills fit: grounded in `activation_rule`, `state_rule`, and benchmark-style key points.
5. How to use them: steps from `execution_rule.intervention_steps`.
6. Avoid: contraindications and `must_not_do`.
7. Suggested next clinician questions: assessment or clarification questions before intervention.

Boundary wording:

- Use "may consider", "is consistent with", "suggests", and "requires clinician judgment".
- Avoid "diagnose", "treat", "guarantee", "the correct intervention is".
- For crisis content, prioritize risk assessment, local emergency resources, and supervisor escalation.

## Simulation Track

Simulation should be built after the practitioner assistant retrieval path is stable.

### Patient simulator

Inputs:

- scenario summary;
- expected key points;
- state cluster / move cell;
- selected or hidden target skills;
- persona constraints;
- risk state.

Behavior:

- answer trainee questions as the patient;
- reveal information gradually;
- maintain emotional and cognitive consistency;
- avoid volunteering all key points at once;
- trigger escalation behavior when high-risk cues are present.

### Practitioner simulator

Use mainly as a reference-response generator or training exemplar. It should be skill-guided, not free-form direct LLM counseling. Generated examples must be labeled as educational references, not clinical gold standards.

## Trainee Evaluation Track

The trainee-evaluation module should treat the skill library as a rubric source.

Input:

```text
trainee-patient dialogue transcript
optional case description
optional supervisor notes
```

Evaluation steps:

1. Extract patient state, goals, context, and risk cues.
2. Retrieve expected skill bundle.
3. Detect trainee moves in the transcript.
4. Compare observed moves with expected key points and retrieved skill rules.
5. Score strengths, omissions, contraindications, and safety handling.
6. Produce educational feedback with concrete alternative wording.

Suggested rubric:

- problem understanding;
- empathy and validation;
- clarification/formulation;
- therapeutic action fit;
- safety/risk assessment;
- actionability;
- boundary and ethics;
- missed opportunities;
- unsafe or over-directive statements.

This track is especially relevant to the brain hospital data, but it requires strict de-identification, access control, and expert review before publication.

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
- `recommend_bundle(query)` returning skill IDs and reasons.
- reproducible config under `configs/`.

Validation:

- run on 20 benchmark case-view items;
- verify all returned skill IDs exist;
- log latency and retrieval profile.

### M1: Practitioner assistant demo

Deliver:

- Hermes-accessible `recommend_bundle` and `get_skill`.
- practitioner-facing response template.
- audit log per query.

Validation:

- run on 50 benchmark items;
- report primary hit@10, bundle recall@10, key-point coverage, and safety notes.

### M2: Full benchmark run

Deliver:

- BM25, dense, hybrid, and metadata-aware profiles.
- results for client and case views.
- summary JSON and paper-ready table.

Validation:

- use fixed current Chinese library checksums;
- separate exact skill-ID recall from key-point coverage.

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

1. Revise `paper.tex` to foreground practitioner-facing Hermes integration.
2. Keep Experiment 1 as scenario-based skill selection.
3. Reframe Experiment 2 as practitioner-copilot response planning and explanation.
4. Add trainee evaluation and simulation as extension applications, not current evidence.
5. Implement the backend before touching Hermes core.
