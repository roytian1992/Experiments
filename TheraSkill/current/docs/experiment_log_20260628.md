# Experiment Log 2026-06-28

## Scope
- Project: `/vepfs-mlp2/c20250513/241404044/users/roytian/TheraSkillAgent`
- Purpose: Skill-library schema cleanup and LLM-based duplicate `skill_name` disambiguation.
- Status: Completed for Chinese `data/skill_library`.

## Trajectory

### 2026-06-28 02:19 CST - LLM Skill Name Deduplication
- Goal: Remove duplicate `skill_name` values in `data/skill_library/skills.jsonl` using LLM-generated Chinese names, without restoring `_zh` duplicate-name fields.
- Starting point: `52070` skill rows; `2440` duplicate name values covering `19367` rows. Example: `sk_000033` and `sk_000034` both used `拖延行为的认知-行为干预`.
- Actions taken: Added `scripts/deduplicate_skill_names.py`; generated an auditable mapping at `reports/skill_name_dedup/20260628-015626/rename_mapping.jsonl`; used `json_repair` for minor LLM JSON formatting repair; applied the mapping to `data/skill_library/skills.jsonl`; rebuilt `data/skill_library/skills_retrieval_index.jsonl`; updated `data/skill_library/manifest.json` and `checksums.sha256`.
- LLM details: `Qwen3-235B-FP8` via `http://127.0.0.1:8001/v1` and `http://127.0.0.1:8002/v1`; `64` workers; chunk size `4`; LLM input excluded stable IDs and trace/source identifiers.
- Decisions or rationale: First-pass LLM names were generated for duplicate rows only. A second-stage LLM collision pass resolved names duplicated across chunks and names colliding with previously unique skill names.
- Verified facts: Final `skills.jsonl` row count `52070`; unique `skill_name` count `52070`; duplicate name values `0`; duplicate rows `0`; retrieval-index name mismatches `0`; checksums verified for README, manifest, skills, retrieval index, and tree nodes.
- Outputs or changed paths: `data/skill_library/skills.jsonl`, `data/skill_library/skills_retrieval_index.jsonl`, `data/skill_library/manifest.json`, `data/skill_library/checksums.sha256`, `scripts/deduplicate_skill_names.py`, `reports/skill_name_dedup/20260628-015626/qa_summary.json`.
- Backup: Original skill file copied to `data/skill_library/skills.before_name_dedup_20260628-021837.jsonl`.
- Example result: `sk_000033` -> `认知共情调节拖延行为`; `sk_000034` -> `结构化管理拖延与情绪`.
- Current state: Chinese formal skill library has no `_zh` fields and no duplicate `skill_name` values.

### 2026-06-28 03:45 CST - English Skill Translation Resume And Residual-CJK Repair
- Goal: Continue generating the English `data_en/skill_library/skills.jsonl` from the updated Chinese `data/skill_library/skills.jsonl`.
- Starting point: Existing English output had `1549` completed rows out of `52070`; the tmux run `translate_en_skills` using `64` workers was stalled on one residual Chinese phrase in a single translated field.
- Problem observed: The prior residual-CJK repair loop repeatedly failed on source `对"孩子的错都是妈妈的错"这一观念的困惑和认同`, with invalid output still containing `认同`; output row count stayed at `1549`.
- Actions taken: Stopped the stalled tmux session, updated `scripts/translate_to_english.py` so residual-CJK handling uses LLM single-field repair from the original Chinese source plus forbidden residual spans, then falls back to whole-field English-only rewrite and indexed single-field translation. No deterministic glossary or non-LLM translation fallback was added.
- Verification: `python -m py_compile scripts/translate_to_english.py` passed. A direct repair call for the stalled source returned `confusion and agreement with the idea that "children's mistakes are all the mother's fault"` with `contains_cjk=False`.
- LLM details: `Qwen3-235B-FP8` via `http://127.0.0.1:8001/v1` and `http://127.0.0.1:8002/v1`; API key from local default `token-abc123`; resumed with `64` workers in tmux session `translate_en_skills`.
- Command: `/vepfs-mlp2/c20250513/241404044/users/roytian/anaconda3/bin/python scripts/translate_to_english.py --output-root data_en --only skill_library.skills --workers 64`
- Outputs and logs: Output path `data_en/skill_library/skills.jsonl`; active log `reports/translate_en_skills_20260628-034551.log`; previous stalled log `reports/translate_en_skills_20260628-034205.log`.
- Current state: The resumed job started from the existing `1549` rows and began writing additional rows.

### 2026-06-28 10:09 CST - English Skill Translation Repair Prompt Tightening
- Goal: Keep the English skill-library translation running after residual-CJK repair warnings became dense around short Chinese spans.
- Starting point: The resumed output reached `3795` rows, but `reports/translate_en_skills_20260628-034551.log` showed repeated repair failures on spans such as `陪伴`, `炫耀`, `侥幸`, `阶段性`, and `认同`.
- Actions taken: Stopped the active tmux session, tightened `scripts/translate_to_english.py` repair prompts to request ASCII English JSON, and removed invalid prior translations from the repair prompt payload so the model sees only the original Chinese source plus forbidden residual spans.
- Verification: `python -m py_compile scripts/translate_to_english.py` passed. Five direct LLM repair samples covering the observed spans all returned English-only outputs with `contains_cjk=False`.
- LLM details: `Qwen3-235B-FP8` via `http://127.0.0.1:8001/v1` and `http://127.0.0.1:8002/v1`; resumed with `64` workers in tmux session `translate_en_skills`.
- Command: `/vepfs-mlp2/c20250513/241404044/users/roytian/anaconda3/bin/python scripts/translate_to_english.py --output-root data_en --only skill_library.skills --workers 64`
- Outputs and logs: Output path `data_en/skill_library/skills.jsonl`; active log `reports/translate_en_skills_20260628-100911.log`; previous warning-heavy log `reports/translate_en_skills_20260628-034551.log`.
- Current state: The resumed job restarted from `3795` completed rows and began writing new rows without immediate warning output in the new log.

### 2026-06-28 22:20 CST - Hermes Practitioner Copilot Reframing
- Goal: Convert the project framing from a generic mental-health support agent into a Hermes-based practitioner-facing copilot plan covering clinician assistance, simulation, and trainee education.
- Starting point: `papers/TheraSkill/paper.tex` described TheraSkill mostly as a generic action layer for mental-health support agents; `docs/hermes_agent_development_plan.md` already proposed a Hermes tool surface but did not foreground the practitioner-assistant product path.
- Inputs checked: Chinese skill-library manifest at `data/skill_library/manifest.json` (`52070` full skill cards, `2245` tree nodes, `9` categories); benchmark manifest at `data/benchmark/manifest.json` (`500` scenarios, `500` client-view items, `500` case-view items); Hermes checkout at `/vepfs-mlp2/c20250513/241404044/users/roytian/hermes-agent-main`; Hermes contribution guidance in `AGENTS.md` and `CONTRIBUTING.md`.
- Actions taken: Added `docs/hermes_practitioner_assistant_plan_20260628.md`; revised `papers/TheraSkill/paper.tex` title, abstract, introduction, background, methodology, Hermes runtime design, experiments, discussion, and conclusion to foreground a practitioner-facing Hermes copilot and to treat simulation/trainee feedback as extensions of the same skill layer.
- Decisions or rationale: Keep TheraSkill domain logic in `TheraSkillAgent`; expose it to Hermes through a narrow local tool or MCP interface rather than embedding therapeutic logic directly into Hermes core. The first product path is practitioner decision support; simulation and trainee transcript evaluation follow after retrieval and explanation are stable.
- Outputs or changed paths: `docs/hermes_practitioner_assistant_plan_20260628.md`; `/vepfs-mlp2/c20250513/241404044/users/roytian/papers/TheraSkill/paper.tex`; regenerated `/vepfs-mlp2/c20250513/241404044/users/roytian/papers/TheraSkill/paper.pdf`.
- Verification: Ran `pdflatex -interaction=nonstopmode -halt-on-error paper.tex` in `/vepfs-mlp2/c20250513/241404044/users/roytian/papers/TheraSkill`; compile succeeded and produced a `13` page PDF.
- Current state: The manuscript now claims library construction, Hermes-oriented runtime design, and planned evaluation protocol, but still avoids claims about final retrieval results, deployed copilot performance, clinical efficacy, or real-world deployment safety.
- Next steps: Implement `theraskill_agent/` backend with `recommend_bundle`, `get_skill`, and `explain_plan`; then expose it through a Hermes-compatible local tool or MCP server and run fixed benchmark profiles on both client and case views.

### 2026-06-28 17:36 CST - Practitioner Assistant Scenario Evaluation Reframing
- Goal: Replace the prior Exp1/Exp2 framing with one unified first application scenario: a practitioner-facing clinical assistant.
- Starting point: `papers/TheraSkill/paper.tex` still framed evaluation as two experiments, including scenario-based skill selection and skill-guided practitioner guidance. The plan document still mentioned user-visible skill IDs, hit@10, bundle recall, client/case views, and Exp1/Exp2 next steps.
- Actions taken: Revised `/vepfs-mlp2/c20250513/241404044/users/roytian/papers/TheraSkill/paper.tex` so the evaluation section is now `Scenario-Based Evaluation Plan` with `Scenario 1: Practitioner-Facing Clinical Assistant`. Revised `docs/hermes_practitioner_assistant_plan_20260628.md` to add a `Scenario 1 Evaluation` section and to remove skill-ID recall, retrieval-tag overlap, move-type coverage, and action-function labels as reported evaluation targets.
- Decisions or rationale: Both retrieval and generation are layers of the same physician-assistant scenario. The evidence layer evaluates whether retrieved skill-card content covers key points and safety evidence. The guidance layer evaluates the final physician-facing note with LLM-as-judge plus expert calibration. Hidden reference skills may support benchmark construction and audit, but model-facing and judge-facing inputs should not expose skill IDs, source IDs, candidate rule IDs, or provenance trace fields.
- Verified facts: Current skill-card schema in `TheraSkillAgent/data/skill_library/skills.jsonl` has `52070` rows and no `move_type` or `therapist_moves` field; original `move_type` exists only in early extraction outputs under `TheraSkill/data_work/outputs/therapist_moves_*`. Current benchmark manifest still has `500` scenarios and `500` answer-key rows per view, but the paper now frames the first model-facing evaluation as the practitioner-assistant scenario rather than a client-view task.
- Outputs or changed paths: `/vepfs-mlp2/c20250513/241404044/users/roytian/papers/TheraSkill/paper.tex`; `docs/hermes_practitioner_assistant_plan_20260628.md`; this log entry.
- Current state: Paper and plan now align on one first scenario: case scenario -> retrieved skill-card evidence -> physician-facing guidance note. Formal runtime metrics remain planned, not reported.
- Next steps: Compile the paper, search for residual Exp1/Exp2/ID-recall wording, then refresh the small `~/Experiments/TheraSkill/current` backup if a push is requested.

### 2026-06-28 17:48 CST - Scenario 2/3 Evaluation Design
- Goal: Add evaluation designs for the simulation and trainee-feedback scenarios without duplicating the practitioner-assistant retrieval/generation metrics.
- Starting point: `docs/hermes_practitioner_assistant_plan_20260628.md` described simulation and trainee feedback as tracks, but did not define scenario-specific evaluation objects, inputs, metrics, or judge/expert roles. `/vepfs-mlp2/c20250513/241404044/users/roytian/papers/TheraSkill/paper.tex` mentioned these analyses only as supporting work.
- Actions taken: Expanded the plan document with `Scenario 2 Evaluation: Training Simulation` and `Scenario 3 Evaluation: Trainee Transcript Feedback`. Added a concise `Scenarios 2 and 3: Simulation and Trainee Feedback` subsection to the paper.
- Decisions or rationale: Scenario 2 evaluates simulated interaction behavior rather than retrieved evidence, with metrics for case fidelity, progressive disclosure, interaction realism, safety-trigger behavior, and controllability. Scenario 3 evaluates educational feedback reports for trainee transcripts, with metrics for issue detection, safety-gap detection, transcript grounding, educational usefulness, and overreach control.
- Outputs or changed paths: `docs/hermes_practitioner_assistant_plan_20260628.md`; `/vepfs-mlp2/c20250513/241404044/users/roytian/papers/TheraSkill/paper.tex`; this log entry.
- Current state: The project now has three scenario-level evaluation designs: practitioner-facing assistant, training simulation, and trainee transcript feedback.
- Next steps: Compile the paper and then turn each scenario into concrete benchmark item schemas and judge prompts.
