# Current Snapshot

- Updated: 2026-06-28 22:20 CST
- Source project: `/vepfs-mlp2/c20250513/241404044/users/roytian/TheraSkillAgent`
- Paper project: `/vepfs-mlp2/c20250513/241404044/users/roytian/papers/TheraSkill`
- Hermes checkout inspected: `/vepfs-mlp2/c20250513/241404044/users/roytian/hermes-agent-main`
- Source git state: source project directories are not Git worktrees; this snapshot is backed up through `~/Experiments`.
- Purpose: Latest TheraSkill practitioner-facing Hermes copilot framing, architecture plan, manuscript wording, and trajectory log.
- Inputs: `data/skill_library/manifest.json`, `data/benchmark/manifest.json`, Hermes `AGENTS.md`, Hermes `CONTRIBUTING.md`, Hermes `toolsets.py`, Hermes `tools/registry.py`.
- Counts verified: 52,070 full skill cards; 52,070 retrieval-index rows; 2,245 tree nodes; 9 skill categories; 500 benchmark scenarios; 500 client-view items; 500 case-view items; 500 answer-key rows per view.
- Commands: `pdflatex -interaction=nonstopmode -halt-on-error paper.tex` in `/vepfs-mlp2/c20250513/241404044/users/roytian/papers/TheraSkill`.
- Outputs: `docs/hermes_practitioner_assistant_plan_20260628.md`, `docs/experiment_log_20260628.md`, `paper/paper.tex`, `paper/paper.pdf`.
- Result or metric files: no new benchmark metrics; manuscript PDF compilation succeeded and produced 13 pages.
- Raw data locations: `/vepfs-mlp2/c20250513/241404044/users/roytian/TheraSkillAgent/data/`; raw data are not copied into this snapshot.
- Previous snapshot archived locally at: see `../.last_archive_path`; archive is local and intentionally not added to Git in this commit.
- Notes: Current positioning is practitioner decision support, simulation, and trainee education through a Hermes-accessible TheraSkill action layer. Claims remain bounded to library construction, Hermes-oriented runtime design, and planned evaluation protocol.
