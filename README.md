# A2Knowledge

A schema-driven pipeline that converts official curriculum and certification documents into [A2UI](https://github.com/curtiskrygier/a2ui)-compatible study apps. The pipeline is LLM-driven with a human review gate; the output is a structured JSON payload renderable on any A2UI surface.

## What it does

```
Official curriculum + course content
        ↓  Prompt 0 — Schema Generator
schemas/<id>.yaml   (required competencies, atom type map)
        ↓  Prompt 1 — Extraction
curriculum.md       (competency-anchored Markdown)
        ↓  Prompt 2 — Suitability Checker
gap-report.json     (coverage analysis — human reviews this)
        ↓  Prompt 3 — A2UI Transformer
payload.json        (A2UI block list)
        ↓  make_url.py (in a2ui-catalogue)
Live GAS study app  (shareable URL, no auth required)
```

Full pipeline specification: [SPEC.md](SPEC.md)

## Live example

French DNB (Brevet) 2026 study app — built with this pipeline from official French Government curriculum sources:

`https://script.google.com/macros/s/AKfycbySUgHU2ynyj-GMc9_oR9qiWlwCVrNSeCwFXY_JYExxIldHYnQFfqgo_vE_uUBJg2L7/exec?nav=brevet-2026`

## Running the pipeline

**Prerequisites:**
- An LLM (Claude or Gemini) to run each prompt
- [`a2ui-catalogue`](https://github.com/curtiskrygier/a2ui) cloned alongside this repo — the `scripts/make_url.py` tool lives there

**Steps:**
1. Choose or create a schema in `schemas/` — see `schemas/registry.yaml` for available schemas
2. Run `prompts/00-schema-generator.md` against the official exam blueprint to generate a schema YAML (skip if schema already exists)
3. Run `prompts/01-extraction.md` with your source curriculum document + the schema → outputs `curriculum.md`
4. Run `prompts/02-suitability.md` to validate coverage → review the gap report, edit `curriculum.md`, repeat until clean
5. Run `prompts/03-transformer.md` → outputs `payload.json`
6. Encode: `python3 ../a2ui-catalogue/scripts/make_url.py payload.json`

## Schemas

| Schema | File | For |
|---|---|---|
| `national-education/fr/dnb-2026` | `schemas/national-education/fr/dnb-2026.yaml` | French DNB (Brevet) — all 5 subjects |
| `pro-cert` | `schemas/pro-cert.yaml` | Generic professional cert template (AWS, GCP, PMP…) |

Add a new schema by copying `schemas/pro-cert.yaml` and filling in `required_competencies` from the official exam blueprint.

## Repo structure

```
a2knowledge/
├── SPEC.md                          ← Pipeline spec and curriculum.md format reference
├── prompts/                         ← Four LLM prompts for the pipeline
├── schemas/                         ← Schema YAMLs defining "complete" per qualification
│   ├── registry.yaml                ← Index of all available schemas
│   ├── national-education/          ← Academic schemas (DNB, etc.)
│   └── pro-cert.yaml                ← Generic pro cert template
├── diagrams/                        ← D2 architecture diagrams
│   ├── pipeline.d2 / pipeline.svg   ← Knowledge Catalogue ingestion pipeline
│   └── ard-architecture.*           ← ARD discovery layer diagram
├── brevet-2026-*.curriculum.md      ← Live French DNB curriculum files (5 subjects)
├── examples/                        ← Reference curriculum.md for a single subject
├── article.md / article.yaml        ← Blog article: ARD, A2UI, and this pipeline
├── media/                           ← Cover images and media catalogue entries
└── screenshots/                     ← Rendered app screenshots
```

## Related

- [A2UI Catalogue](https://github.com/curtiskrygier/a2ui) — the atom vocabulary and renderers this pipeline feeds
- [SPEC.md](SPEC.md) — full format spec, LMS interoperability notes, schema system documentation
