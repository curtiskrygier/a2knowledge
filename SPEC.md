# A2UI Knowledge Catalogue — Specification

**Version:** 1.0 | **Date:** 2026-06-21

> Canonical spec for the A2UI Knowledge Catalogue format and ingestion pipeline.

---

## The Core Idea

Most study apps are one-off builds: someone hand-crafts content, exports to a format, and the app is born. The curriculum changes, the app rots.

The A2UI Knowledge Catalogue inverts this. The **curriculum is the source of truth** — structured, versioned, and schema-aware. The A2UI app is a *rendered view* of that curriculum, regeneratable on demand.

The pipeline: **Source → curriculum.md → gap report → payload.json → live app**. The only step that requires human judgment is the gap report review. Everything else is LLM-driven and schema-constrained.

See `diagrams/pipeline.svg` for the full architecture diagram.

---

## The curriculum.md Format

Every curriculum file is a single Markdown document with:

### Frontmatter (YAML)
Uses **Schema.org** vocabulary so the frontmatter is portable across systems that understand Schema.org `Course` (Moodle, Canvas, edX all use this vocabulary, though import compatibility requires additional tooling per-system):

```yaml
---
id: brevet-2026-maths           # kebab-case unique identifier
type: Course                     # Schema.org type
schema: fr-dnb                   # parser/validator behaviour switch
name: "Mathématiques — DNB 2026"
educationalLevel: "Collège 3ème" # Schema.org
provider:
  name: "Ministère de l'Éducation Nationale"
competencyFramework: fr-socle-commun-c4  # CASE reference
source: source_dnb2026_automatismes.pdf

required_competencies:           # What a COMPLETE doc must cover
  - id: maths-geometrie-pythagore
    label: "Pythagore — direct, réciproque, contraposée"
    weight: high                 # high | medium | low

exam:                            # ISO 8601 durations
  duration: PT2H
  total_points: 20
  parts:
    - id: automatismes
      name: Automatismes
      duration: PT20M
      points: 6
      constraints: [no-calculator, timed-removal]

render:                          # A2UI-only — ignored by non-A2UI consumers
  hub_label: "📐 Maths"
  hub_color: "#6366f1"
---
```

### Section types
Typed headings tell the parser what A2UI atom to generate:

| Tag | Atom | Usage |
|---|---|---|
| `{#drill}` | `brevet_automatismes` | Tables of facts for timed practice |
| `{#concept}` | `flashcard_deck` | Heading = front, body = back |
| `{#glossary}` | `flashcard_deck` | Bullet `- **term** : definition` |
| `{#timeline}` | `brevet_timeline` | h3 entries: `### YYYY \| Title` |
| `{#method}` | `steps` | Numbered steps |
| `{#key_takeaways}` | `key_takeaways` | Bullet list of key facts |
| `[!PIÈGE]` callout | `knowledge_check` | Common trap + correction |

### Competency anchors
HTML comment before each section links content to a required competency:
```
<!-- competency: maths-geometrie-pythagore -->
## Théorème de Pythagore {#concept .weight-high}
```
This is the machine-readable link that enables gap detection. Invisible in rendered Markdown, trivially parseable by LLMs and Python.

---

## The 4-Prompt Pipeline

Two source documents feed the pipeline with different roles:
- **Official Curriculum** (exam blueprint PDF) → defines *what* must be covered → becomes the schema
- **Course Content** (textbook, study manual) → explains *how* to cover each topic → becomes curriculum.md

### Prompt 0 — Schema Generator
**Input:** Official curriculum document
**Task:** LLM extracts all required competencies, exam structure, and forbidden topics → outputs schema YAML
**Run once per qualification.** Re-run when the official programme is updated.
**Output:** `schemas/<schema-id>.yaml`

See: `prompts/00-schema-generator.md`

### Prompt 1 — Extraction
**Input:** Official curriculum + course content + schema YAML (all three)
**Task:** LLM reads course content, emits complete `curriculum.md` with frontmatter, typed sections, and competency anchors. Uses the schema as the target list and the curriculum doc as the "what to cover" guide.
**Key rule:** Every `required_competency` from the schema must have a section — even a thin placeholder.

See: `prompts/01-extraction.md`

### Prompt 2 — Suitability Checker
**Input:** `curriculum.md` + schema YAML
**Task:** LLM cross-references every `<!-- competency: id -->` anchor against `required_competencies`. Outputs structured JSON:

```json
{
  "coverage": [
    {"id": "maths-geometrie-pythagore", "status": "full", "note": ""},
    {"id": "maths-stats-probabilites", "status": "partial", "note": "Arbres de probabilité missing"}
  ],
  "accuracy_issues": [],
  "quality_issues": [],
  "overall": "needs_review",
  "recommended_actions": ["Add probability trees section"]
}
```

**Human gate:** Review gap report. Edit curriculum.md if needed. Re-run until `overall: ready`.

See: `prompts/02-suitability.md`

### Prompt 3 — A2UI Transformer
**Input:** Approved `curriculum.md` + schema YAML
**Task:** LLM maps each section to its A2UI atom type, builds the hub structure, outputs valid `payload.json`
**Output:** `payload.json` — encode with `make_url.py` from the [a2ui-catalogue](https://github.com/curtiskrygier/a2ui) repo:

```bash
python3 ../a2ui-catalogue/scripts/make_url.py payload.json
```

See: `prompts/03-transformer.md`

---

## Schema System

Schemas define what "complete" means for a given qualification type. Stored in `schemas/`.

| Schema | File | For |
|---|---|---|
| `national-education/fr/dnb-2026` | `schemas/national-education/fr/dnb-2026.yaml` | French DNB (brevet) — all 5 subjects |
| `pro-cert` | `schemas/pro-cert.yaml` | Generic professional cert (AWS, GCP, PMP…) |

Each schema carries:
- `required_competencies` per subject/domain — the ground truth for gap detection
- `forbidden` list — topics that look plausible but are out of scope
- `atom_type_map` — section tag → A2UI atom type
- `render_strategy` — `subject_hub` (academic) or `domain_hub` (pro cert)

**Adding a new schema:** Copy `schemas/pro-cert.yaml`, fill in `required_competencies` from the official exam blueprint.

---

## Standards Vocabulary

The format borrows vocabulary from established standards for interoperability:

| Standard | Used for |
|---|---|
| **Schema.org** `Course` | frontmatter field names (`type`, `educationalLevel`, `provider`) |
| **1EdTech CASE** | competency ID scheme and `required_competencies` structure |
| **QTI-lite** | `[!PIÈGE]` → question/correct/explanation pattern |
| **ISO 8601** | exam durations (`PT2H`, `PT20M`) |

The `render:` block is the only A2UI-specific section — non-A2UI consumers ignore it. The vocabulary alignment means `curriculum.md` files are portable across systems that understand Schema.org and CASE, though direct LMS import (Moodle, Canvas) requires format translation; no one-click import exists.

---

## Why This Matters for A2UI

The standard A2UI demo is: *"here's a JSON payload, here's the rendered atom."*

The Knowledge Catalogue demo is: *"here's a national curriculum, here's the gap analysis, here's the study app — generated end-to-end from structured knowledge."*

A2UI becomes not just a UI renderer but a **knowledge rendering platform**:
- The curriculum.md is the knowledge graph node
- The schema YAML is the validation vocabulary
- The A2UI atoms are the rendered views
- The suitability checker is the quality gate

The pattern scales to any structured qualification — the brevet is the first shipped example. Each cert gets a schema YAML. Each subject gets a curriculum.md. The renderer is the same.

---

## File Structure

```
a2knowledge/
├── SPEC.md                                  ← this file
├── diagrams/
│   ├── pipeline.d2 / pipeline.svg           ← ingestion pipeline (D2 source + rendered)
│   └── ard-architecture.d2 / .svg           ← ARD discovery layer
├── schemas/
│   ├── registry.yaml                        ← index of all schemas
│   ├── national-education/fr/dnb-2026.yaml  ← DNB required competencies per subject
│   └── pro-cert.yaml                        ← Generic pro cert schema + examples
├── prompts/
│   ├── 00-schema-generator.md               ← Prompt 0: official curriculum → schema YAML
│   ├── 01-extraction.md                     ← Prompt 1: source → curriculum.md
│   ├── 02-suitability.md                    ← Prompt 2: validate coverage
│   └── 03-transformer.md                    ← Prompt 3: curriculum.md → payload.json
├── brevet-2026-*.curriculum.md              ← Live French DNB curriculum (5 subjects)
└── examples/                                ← Additional reference curriculum files
```

---

## Live Example

The brevet 2026 app was built with this pipeline:
- Schema: `national-education/fr/dnb-2026`
- Curriculum files: `brevet-2026-maths.curriculum.md` and siblings in repo root
- Live: `https://script.google.com/macros/s/AKfycbySUgHU2ynyj-GMc9_oR9qiWlwCVrNSeCwFXY_JYExxIldHYnQFfqgo_vE_uUBJg2L7/exec?nav=brevet-2026`

The brevet audit identified 17 gaps before the schema was written. The `dnb-2026.yaml` schema now encodes those lessons — run the suitability checker on the brevet curriculum.md and it will catch all 17 automatically.
