# Physical Setting Playbook (ERIS text → Phase I narrative)

## Goal
Generate a defensible Phase I ESA **Physical Setting** section from ERIS reports converted to text.

Deliverables:
1) `index.json` — structured map of the ERIS text by report and section
2) `fact_sheet.json` — canonical extracted facts + evidence
3) `qa_report.json` — completeness/conflict/wording checks
4) `physical_setting.md` — final narrative (templated)

Core principle: **extraction is separate from writing**.
- Extraction produces facts + evidence.
- Writing renders those facts into a fixed template.

---

## Inputs
Expected inputs are plain text files, typically one per ERIS report, e.g.:
- Physical Setting Report (ERIS).txt
- Topographic Maps Report (ERIS).txt
- Aerial Photographs Report (ERIS).txt
- City Directories Report (ERIS).txt

The pipeline should also accept a single combined text file.

---

## Step 1 — Create an “Index” (structured text map)

### What “index” means
An index is a deterministic lookup structure:
- report_type → sections → exact text blocks (+ optional offsets)

This allows downstream steps to say:
- “Floodplain statements MUST come from the FEMA section block”
- “Soils summary MUST come from the SSURGO/Soils section block”

### Index format (recommended)
`out/index.json`:

```json
{
  "reports": [
    {
      "report_type": "Physical Setting Report",
      "source_file": "Physical Setting Report (ERIS).txt",
      "sections": [
        {
          "section_name": "Topography",
          "text": "...",
          "start_line": 120,
          "end_line": 210
        }
      ]
    }
  ]
}
```

Offsets are optional but strongly recommended for traceability.

### How to build the index (rules)
1) Normalize text:
   - unify line endings
   - collapse repeated whitespace (but keep line numbers stable if you track them)
   - keep a raw copy for evidence snippets

2) Detect report boundaries:
   - Prefer file-level separation if files are already per-report.
   - If combined file: split by known report headers (case-insensitive), e.g.:
     - "PHYSICAL SETTING REPORT"
     - "TOPOGRAPHIC MAPS REPORT"
     - "AERIAL PHOTOGRAPHS REPORT"
     - "CITY DIRECTORIES REPORT"

3) Within each report, detect section boundaries:
   - Use headings when present (all caps lines, colon-terminated headers, numbered headers).
   - Also support keyword-based sectioning when headings are inconsistent.

### Standard section taxonomy
Use these canonical section names (map whatever ERIS calls it into these buckets):

- Topography
- Surface Water / Drainage
- Soils (SSURGO)
- Geology / Bedrock (if present)
- Hydrogeology / Groundwater (if present)
- Floodplains (FEMA)
- Wetlands (NWI / state)
- Aerials (by year if possible)

If you can’t confidently split, store a section `"Other"` and rely on keyword retrieval within the report.

---

## Step 2 — Extract a Physical Setting Fact Sheet (canonical JSON)

### Fact sheet purpose
A single normalized record that the narrative can be generated from without rereading ERIS text.

### Fact sheet schema (high-level)
Store:
- value(s)
- evidence list
- status (explicit / derived / unknown)
- notes/conflicts

Recommended shape for each field:

```json
{
  "value": "SW",
  "status": "explicit",
  "evidence": [
    {
      "report_type": "Physical Setting Report",
      "section_name": "Topography",
      "snippet": "Slope direction: SW",
      "start_line": 155,
      "end_line": 155
    }
  ]
}
```

### Minimum required fields
Topography
- site_elevation_ft
- slope_direction
- (optional) elevation_range_1mi_ft (min/max) if present in text

Surface water / drainage
- onsite_water_features (list)
- nearby_named_waterbodies (list)
- drainage_description (short narrative-friendly phrase)

Soils / geology
- soil_map_units (name, percent if available, drainage class, hydrologic group)
- soil_summary (1–2 sentences derived from explicit soil descriptors)
- bedrock_depth (if explicitly stated)

Hydrogeology
- estimated_groundwater_flow_direction (explicit if present; otherwise derived)
- basis (required if derived)

Floodplains
- fema_zone (if stated)
- mapped_in_100yr_floodplain (true/false/unknown)
- notes (e.g., “based on FEMA mapping in ERIS”)

Wetlands
- nwi_or_mapped_wetlands_on_site (true/false/unknown)
- state_wetlands_mapped_on_site (true/false/unknown)

Data quality
- conflicts (list)
- assumptions (list)
- unknowns (list)

### Extraction precedence (when multiple sources exist)
1) Physical Setting Report text (usually most direct)
2) Topographic report text (for topo/map-derived statements already described in text)
3) Aerial report text (for feature identification already described in text)
4) Anything else (City directories usually not physical setting; ignore unless it explicitly mentions landform/water)

---

## Step 3 — Apply Interpretation Rules (controlled derivations)

All derived statements must:
- set `status: "derived"`
- include a `basis` field
- be written in narrative with “estimated” language

### Interpretation rules (must be consistent)
**Rule A — Groundwater flow**
- If ERIS explicitly states groundwater flow direction, use it (explicit).
- Else derive it from slope/topography if slope_direction exists:
  - estimated_groundwater_flow_direction = slope_direction
  - basis = "Estimated based on topographic gradient/slope direction; no site-specific well data reviewed."
- If no slope direction: set unknown.

**Rule B — “Not mapped” vs “Not present”**
- If ERIS says “no mapped wetlands” → narrative: “No wetlands are mapped on the Site in reviewed sources.”
- Do NOT say “wetlands are not present” unless explicitly confirmed by site inspection text (outside ERIS) or a statement that confirms absence.

**Rule C — Dominant soils selection**
If soil units include percent coverage:
- dominant_soils = top 2–4 by percent OR all units ≥ 10%
If no percent:
- list all mentioned units, but do not imply dominance.

**Rule D — Surface drainage descriptors**
Prefer restrained phrasing:
- “Surface drainage is expected to generally follow local topography toward …”
Avoid engineering claims (“will infiltrate”) unless explicitly stated.

**Rule E — Conflicts**
If two values disagree (e.g., different elevation values):
- keep both in `conflicts`
- choose the one from the more authoritative / explicit section (usually Physical Setting)
- flag in QA

Interpretation rules should be codified in `rules/interpretation_rules.yaml` (ideal) so they’re testable and versioned.

---

## Step 4 — Render narrative using a locked template

### Template philosophy
Every sentence maps to:
- one or more fact_sheet fields
- and never introduces new facts

Use a Jinja2 template stored at `templates/physical_setting.md.j2`.

### Required headings (match Phase I style)
1) Topography
2) Surface Water
3) Soils and Geology
4) Hydrogeology
5) Floodplains
6) Wetlands

### Recommended template logic (summary)
- If a field is null/unknown: say “Information was not available in the ERIS records reviewed.”
- If derived: include “estimated” and the basis.
- If “not mapped”: explicitly use “not mapped/identified in reviewed sources”.

Example clauses (illustrative):
- Topography:
  - “The Site is located at approximately {{site_elevation_ft}} feet above mean sea level.”
  - “Topography generally slopes toward the {{slope_direction}}.”
- Floodplains:
  - “Based on FEMA mapping reviewed in the ERIS report, the Site is {{floodplain_statement}}.”
- Wetlands:
  - “Based on mapped wetland resources (e.g., NWI/state mapping), wetlands are {{wetlands_statement}}.”

Keep the template stable; change logic via facts and rules.

---

## Step 5 — QA checks (automated)

### QA outputs
Create `out/qa_report.json` with:
- errors (must fix)
- warnings (review)
- info (notes)

### QA checks (required)
**Traceability**
- Every non-null field has at least one evidence entry.
- Evidence snippets are verbatim fragments from indexed text.

**Wording compliance**
- Ban absolute wording unless explicitly supported:
  - banned: “no X exists”, “X is absent”, “will infiltrate”, “definitively”
  - required replacements: “not mapped/identified in reviewed sources”, “estimated”, “based on …”

**Completeness**
- Narrative must contain all 6 headings.
- Fact sheet must include minimum required fields (even if null).

**Conflicts**
- If `conflicts` not empty: QA warning with pointers.

**Determinism**
- Running pipeline twice on same fixtures produces identical JSON narrative outputs (byte-for-byte).

---

## Implementation notes (recommended modules)

1) `src/indexer.py`
- build_index(input_paths) -> index.json

2) `src/extractors/`
- explicit_field_extractors.py (regex/heuristics)
- soils_extractor.py (table-ish parsing from text)
- floodplain_extractor.py
- wetlands_extractor.py

3) `src/rules.py`
- apply_interpretation_rules(fact_sheet, rules_yaml) -> updated fact_sheet

4) `src/renderer.py`
- render(template, fact_sheet) -> physical_setting.md

5) `src/qa.py`
- run_qa(index, fact_sheet, narrative) -> qa_report.json

6) `src/cli.py`
- CLI to run the pipeline end-to-end

---

## Fixtures & testing guidance

Add fixtures under:
- `tests/fixtures/eris_text/` (small representative samples)
- `tests/fixtures/expected/` (golden outputs for index/facts/narrative)

Tests should validate:
- schema correctness
- evidence presence
- prohibited wording not present
- consistent output

---

## Acceptance criteria (what “good” looks like)
- The narrative reads like a Phase I Physical Setting section.
- Every factual statement in the narrative can be traced back to an evidence snippet in ERIS text.
- Derived statements are clearly labeled “estimated” with basis.
- Outputs are deterministic and pass QA.
