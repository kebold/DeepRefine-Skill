# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Identity

DeepRefine-Skill is an agent skill that refines graphify knowledge graphs using the Reafiner algorithm (8-step state machine: INIT → RETRIEVE → JUDGE → RETRIEVE_KHOP → ABDUCE → REFINE → VALIDATE → APPLY → FINISH). It ships as `deeprefine-cli` on PyPI (v0.1.8).

**Our development plan** is documented in `docs/PROJECT-PLAN.md`. The goal: port the skill from Cursor to GitHub Copilot CLI (P0), build a progressively hardened runtime harness, and optionally adapt to Claude Code (P1). Four phases over ~13 weeks.

## Architecture

The codebase has a clean separation between framework-agnostic Python logic and framework-specific instruction files:

### Framework-agnostic core (`deeprefine_skill/`)

All Python modules operate on local files (`graph.json`, `loop_trace_*.json`, `history.jsonl`) and are independent of any agent framework. The CLI (`cli.py`) exposes these as subcommands that any agent can invoke via Bash.

| Module | Role |
|--------|------|
| `agent_loop.py` | Trace validation against Reafiner control flow (`validate_trace()`). Defines regex patterns (`JUDGE_RE`, `ABDUCTION_RE`, `REFINEMENT_RE`) for structured output parsing. Contains `reafiner_early_exit()` and `reafiner_needs_refinement()` — the two branching rules from the paper. |
| `agent_graph.py` | Applies refinement actions to `graph.json`. Parses `<refinement>` blocks, executes `insert_edge`/`delete_edge`/`replace_node` via `apply_refinement_text()`. Handles node lookup by label/alias with Unicode normalization. Supports `source_file::label` qualified names. |
| `agent_prompts.py` | Verbatim prompt templates from the DeepRefine paper. Three LLM call types: judgement (`<judge>Yes/No</judge>`), error abduction (`<abduction>...</abduction>`), KG refinement (`<refinement>action|action</refinement>`). Also defines constants: `MAX_HOPS_DEFAULT=4`, `MAX_TRIPLE_NUM_BY_STEP=[5,10,15,20]`, `HISTORY_HORIZON_DEFAULT=4`. |
| `action_review.py` | **Harness Level 1.5**: Evidence-aware dry-run review engine. Parses `<refinement>` actions, audits each against `graph.json` nodes/edges, detects ambiguous bare names, greps source files for code evidence, assigns HIGH/MEDIUM/LOW confidence labels. `cmd_apply` enforces the review gate — refuses LOW-confidence actions without `--allow-low-confidence`. |
| `history.py` | JSONL-based query queue. `sync_history_from_memory()` imports from `graphify-out/memory/query_*.md` frontmatter. `pending_queries()` returns unrefined entries deduped by id. `mark_refined()` sets `refined=true`. Query IDs are SHA256 truncated to 16 hex chars. |
| `cli.py` | CLI entry point. Subcommands: `cursor/copilot/gemini install/uninstall`, `history (add|list|sync-memory)`, `index --rebuild`, `refine [--apply]`, `review`, `apply [--allow-low-confidence]`, `loop (init|validate|finish)`. `apply` requires `--trace-file`, runs `validate_trace()` AND `action_review` gate before touching `graph.json`. |
| `paths.py` | Path resolution. `find_project_root()` walks up for `graphify-out/graph.json`. `graphify_paths()` returns the canonical path dict. `find_deeprefine_repo()` locates the paper repo (needed only for Terminal CLI mode). |
| `installers.py` | Multi-platform skill installers. Supports Cursor (`.cursor/skills/deeprefine/`), Copilot CLI (`.github/skills/deeprefine/`), and Gemini CLI (`gemini extensions link/install` + manual copy fallback). Uses shared `_resolve_skill_source()` for bundled-skill resolution. |
| `refine_runner.py` | Terminal CLI mode using FAISS + API/vLLM. Dry-run by default; `--apply` to persist. Not used in agent mode. |
| `adapter_graphify.py` | Graphify data loading + FAISS index. Not used in agent mode. |

### Framework-specific instruction files

| File | Target | Status |
|------|--------|--------|
| `SKILL.md` / `deeprefine_skill/SKILL.md` | Cursor | Existing. YAML frontmatter + Markdown describing the full Reafiner loop with dry-run safety policy. |
| `deeprefine_skill/SKILL_COPILOT.md` | Copilot CLI | **Done (P0)**. Copilot-variant frontmatter (`allowed-tools: shell`, no `disable-model-invocation`), mode detection preamble, dry-run review workflow. Installed via `deeprefine copilot install`. |
| `deeprefine_skill/gemini_extension/` | Gemini CLI | **Done** (ziymeng, PR #2). Full extension directory: `gemini-extension.json`, `GEMINI.md`, `commands/*.toml`, `skills/deeprefine/SKILL.md`. Installed via `deeprefine gemini link/install`. |
| `.claude/skills/deeprefine/SKILL.md` | Claude Code | **To be created (P1)** — emitted from `templates/claude-skill.md.j2` |

Instruction files are generated from a single YAML spec (`deeprefine_skill/spec/`) via Jinja2 templates (`templates/`). There is no IR layer — with only 2 Markdown targets, templates are simpler and sufficient. Run `deeprefine compile --target copilot|claude` to regenerate.

### Control flow (agent mode)

```
User: /deeprefine
  → SKILL.md loaded as system instructions
  → Agent runs: deeprefine history sync-memory
  → Agent loads pending queries from history.jsonl
  → For each query:
      deeprefine loop init --query "..."
      Step 1: graphify query → judge → append to loop_trace
      Steps 2..MAX_HOPS: k-hop expand → judge → append
      If len(history)==1 && answerable: early exit → loop finish
      Else: abduction → refinement → loop validate → review → HARD STOP
      (user approval in follow-up message)
      → apply (with action_review gate) → loop finish
```

The `deeprefine apply` command internally calls `validate_trace()` AND the
`action_review` evidence gate.  It refuses LOW-confidence actions unless
`--allow-low-confidence` is passed.  The `--skip-trace-check` flag bypasses
trace validation but NOT the review gate — SKILL.md explicitly forbids using it.

## Harness Engineering Roadmap

The current harness is Level 0: post-hoc validation (`validate_trace()`). Our plan progressively hardens it:

| Level | What | Where |
|-------|------|-------|
| 0 | Post-hoc trace validation | `agent_loop.py` — exists |
| 1 | Real-time output gates (regex reject + retry, max 3 retries per step) | To build in Phase 2.2 |
| 1.5 | Semantic evidence audit (node existence, ambiguity detection, code grep, duplicate prevention; hard gate in `apply`) | `action_review.py` — exists (PR #1) |
| 2 | Deterministic k-hop expansion (BFS, not LLM) | To build in Phase 2.3 |
| 3 | Hard state machine with guard conditions + `loop next`/`loop state` commands | To build in Phase 2.4 |

### FSM Error Recovery: Three-Layer Defense

CLI agents (Copilot, Claude) may not gracefully recover from harness rejections. We defend at three layers:

1. **Instruction layer**: mirror all FSM rules in the platform instruction file so the LLM knows constraints before acting
2. **Assist layer**: `deeprefine loop next --trace-file ...` tells the LLM exactly what step to execute next — reduces decision burden
3. **Enforce layer**: Python FSM guards reject illegal transitions with **actionable error messages** (current state, expected state, suggested CLI command)

If a platform fails to recover even with actionable errors, that is documented as a framework capability gap — a valid research finding.

All harness levels are framework-agnostic Python — they will be reusable across Copilot CLI and Claude Code without modification.

## Development Commands

```bash
# Install in dev mode
pip install -e .

# Run CLI (equivalent)
deeprefine --help
python scripts/deeprefine.py --help   # without pip install

# Lint (ruff not yet configured in CI — add it)
pip install ruff
ruff check deeprefine_skill/
ruff format --check deeprefine_skill/

# Type check (mypy not yet configured)
pip install mypy
mypy deeprefine_skill/

# Tests (test suite does not exist yet — building it is Phase 2.3)
mkdir -p tests
# pytest tests/ -v
```

No test suite, CI, or pre-commit hooks exist yet. Test coverage is 0%. This is addressed in Phase 2.

## Key Constraints

- The prompt templates in `agent_prompts.py` must stay verbatim from the DeepRefine paper. Changing wording changes LLM behaviour and breaks comparability.
- `validate_trace()` is the reference implementation of Reafiner control-flow rules. Any harness mechanism must be consistent with its checks.
- `graph.json` uses either `"links"` or `"edges"` as the key for relationships — `agent_graph.py` handles both.
- The project root is detected by the presence of `graphify-out/graph.json`, not by git or config files.
- Query IDs are `sha256(query.strip())[:16]` — this is used across `history.py`, `agent_loop.py`, and `cli.py` for dedup and file naming.

## Repository Notes

- `docs/PROJECT-PLAN.md` — current technical plan (authoritative)
- `scripts/deeprefine.py` — lightweight CLI entry point; delegates to `deeprefine_skill.cli:main`
