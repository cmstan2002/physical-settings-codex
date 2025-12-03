# AGENTS.md

## Purpose
This repo builds a repeatable pipeline to draft a Phase I ESA **Physical Setting** section from ERIS reports that have already been converted to text.

Target workflow:
ERIS text → indexed sections → fact sheet JSON → interpretation rules → templated narrative → QA checks → final Physical Setting section

## Read first (required)
Before writing code or editing templates, read:
- docs/physical-setting-playbook.md
- rules/interpretation_rules.yaml (or .md if YAML not present)
- schemas/physical_setting_fact_sheet.schema.json
- templates/physical_setting.md.j2

If any of these files don’t exist yet, create them exactly as described in the playbook.

## Non-negotiable rules
1) **No invention of facts**
   - If a value is not explicitly stated in the ERIS text, store it as null/unknown.
   - Do not “sound confident” when the evidence isn’t there.

2) **Traceability**
   - Every extracted non-null fact MUST include evidence:
     - report_type
     - section_name
     - a short evidence snippet (5–25 words copied from the text)
     - (optional) line/char offsets if available

3) **Defensible language**
   - Use “not mapped/identified in reviewed sources” (NOT “not present”) unless the source explicitly confirms absence.
   - Any inferred items (e.g., groundwater flow) MUST be labeled “estimated” and include the basis.

4) **Determinism**
   - Same input text MUST yield the same output JSON + narrative.
   - Prefer rule-based parsing for explicit fields; use LLM-style extraction only for descriptive parts (if at all).

## Repository conventions
- Source code lives in: `src/`
- Tests live in: `tests/`
- Templates live in: `templates/`
- Schemas live in: `schemas/`
- Rules live in: `rules/`
- Documentation lives in: `docs/`

## Output artifacts (Definition of Done)
An end-to-end run on fixture input must produce:
- out/index.json
- out/fact_sheet.json
- out/qa_report.json
- out/physical_setting.md

## CLI expectation
Implement a CLI that can run the full pipeline from ERIS text.
Example interface (adjust as needed):
- `python -m src.cli run --input tests/fixtures/eris_text/ --out out/`
- or `python -m eris_phase1 run --input ...`

## Verification (run all)
- `python -m pip install -r requirements-dev.txt`
- `ruff check .`
- `mypy src`
- `pytest -q`

If introducing new dependencies, update requirements files and add tests.

## Testing requirements
- Unit tests for:
  - indexing (splitting into report/sections)
  - fact extraction (explicit fields)
  - interpretation rules (derived/inferred fields)
  - rendering (template mapping)
  - QA checks (missing evidence, forbidden wording, conflicts)
- At least one end-to-end test on fixtures that asserts:
  - schema validity
  - required subsections exist in narrative
  - every non-null fact has evidence
  - no forbidden absolute claims

## Coding style
- Small pure functions, minimal side effects.
- Prefer dataclasses / pydantic models for typed structures.
- Keep parsing rules isolated and testable.
