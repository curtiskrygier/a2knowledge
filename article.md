# Another Week, Another Agentic Standard Enters the Stage

There's a pattern forming. If you work anywhere near AI infrastructure right now, you've probably had the experience of picking up where you left off on a Friday only to find that three new protocols, formats, or specifications have arrived before Monday morning coffee.

This week's instalment is a good one. A cross-industry working group — Google, Microsoft, Hugging Face, and contributors from Cisco, GitHub, NVIDIA, Salesforce, and others — has published the **Agentic Resource Discovery (ARD)** specification: an open protocol for how AI agents should discover, evaluate, and connect to resources across the web. Alongside that, Google has published the **Open Knowledge Format (OKF)** — an open specification that formalises something many AI teams have already been doing informally: representing internal knowledge as a directory of markdown files with YAML frontmatter, portable across any system without translation. The problem OKF targets is real and widespread: every organisation building on AI is solving the same context-assembly problem from scratch, with knowledge scattered across incompatible wikis, proprietary metadata catalogs, and code comments no agent can reliably reach. OKF proposes a vendor-neutral format — just files, just markdown, no SDK required — so that knowledge produced by one system can be consumed by any agent without a bespoke integration. And underpinning all of it, A2A and MCP continue to mature as the plumbing layer for how agents actually talk to each other and to tools.

This post is my attempt to make sense of where these fit — not in the abstract, but in the context of things I've actually been building.

---

## My Conviction: Useful for Humans, Declarative for Agents

Before I get into the specifics, the thread that runs through everything here is a conviction I've been stress-testing in my own work:

> **AI needs to be useful for humans and declarative for AI agents.**

These aren't two separate goals. They're the same goal expressed at two different layers of the stack. If the human-facing output isn't genuinely useful, the agent hasn't done its job. If the agent-facing specification isn't declarative and machine-readable, you haven't built an agent workflow — you've just written more glue code.

This is the lens through which I've been approaching everything below.

---

## What ARD Actually Proposes

ARD is the industry's answer to a practical problem: agents need to find things. Not web pages — *resources*. APIs, tools, knowledge bases, render surfaces. The spec proposes two complementary paths:

![ARD Architecture](https://raw.githubusercontent.com/curtiskrygier/a2knowledge/master/diagrams/ard-architecture.svg)

**The Catalog layer** is the publish side. Any resource owner can host an `ai-catalog.json` at a well-known path on their domain. No registration, no central authority. The catalog declares what you have, what it does, and how to invoke it.

**The Registry layer** is the discovery side. An optional indexed service that crawls catalogs, ranks results by agent intent, and returns structured matches. Think of it as the difference between putting a sign on your door versus being listed in a directory that agents are actively searching.

The agent at the bottom consumes from either or both, verifies identity via Trust Manifests, and invokes resources using whatever protocol they declare — A2A, MCP, or plain API.

What I found interesting about ARD when I read it: it's not trying to replace anything. It's a discovery and capability description layer that sits *above* the transport protocols. A2A and MCP handle *how* agents communicate. ARD handles *how they find what to communicate with*.

---

## Where My Work Fits In: The A2UI Catalogue

A few months ago I started building what I've been calling an **A2UI catalogue** — a vocabulary of UI atoms that render across different surfaces. The key insight from [Google's A2UI work](https://a2ui.org) was that if you express UI declaratively (as a schema describing *what to show* rather than *how to show it*), an agent can compose a useful, rendered experience without needing to understand any frontend framework.

My catalogue currently covers four surfaces: Google Apps Script, web/blog (powering this site), Google Meet stage panels, and Google Chat cards. Each atom in the catalogue is tagged with which surfaces can render it — which matters, because surface constraints are real. The GAS iframe environment, for example, has limitations that some atoms simply don't fit.

In ARD terms, my A2UI catalogue is a **catalog entry** that describes render capabilities. An agent that discovers it knows: *this resource can take a structured payload and return a rendered learning app, and it works on GAS, and here's the URL to invoke it.*

---

## The Apps Script Renderer: A Pattern Worth Pausing On

This deserves a moment because the pattern it enables is genuinely unusual.

Google Apps Script has a constraint that, if you look at it a certain way, becomes a feature: the execution environment is sandboxed, the output is served from Google's infrastructure, and the URL is stable and shareable. What I found is that you can encode an entire app's content as a base64 payload in the query string. The published GAS app decodes it and renders the relevant atoms.

The result:

- **One published app** — single deployment, single URL, always current
- **N different experiences** — each driven by a different base64-encoded payload in the URL parameter
- **No backend** — the "database" is the URL itself
- **Rendering in under 10 seconds** — once the knowledge payload is ready, encoding it and getting a live GAS URL takes under 10 seconds. Give an agent the atom vocabulary from the catalogue, ask it to compose a payload, encode it, done.

This works well for apps up to roughly 8K characters of encoded content. Beyond that, you'd publish a dedicated URL for that specific payload — still declarative, just not sharing the same entry point.

The important architectural point: **Apps Script is just a service.** The A2UI catalogue is what declares it as a capability, describes what it can render, and tells an agent how to invoke it. Swap out GAS for another surface tomorrow, and the catalogue entry changes — the agents consuming it don't need to.

---

## A2Knowledge: Testing the ARD Principle with a Second Catalogue

I wanted to push the ARD concept further: what happens when you have *two* declarative catalogues and an agent that can compound them?

Knowledge transfer was the use case I landed on. The inputs are always fairly standard — curriculum, course content, certification format — and the outputs have real human value. It's also a domain where the friction of current tools is obvious: PDFs, scattered official documents, per-student manual effort.

So I built an **A2Knowledge catalogue**: a schema-driven pipeline that takes official curriculum sources, extracts competency-mapped knowledge as a structured markdown, validates coverage against a schema, and transforms it into an A2UI payload. The atoms the agent can use are the ones declared in the A2UI catalogue. The surface constraints are already baked in. The agent doesn't need to reason about what's renderable — the catalogue tells it.

The format — competency-anchored markdown with YAML frontmatter, schema-validated and version-controlled — is what Google is calling OKF. The A2Knowledge pipeline arrived at the same structure independently, which suggests the pattern is load-bearing rather than accidental.

---

## The Brevet Test Case

My daughter is sitting the French **Brevet** — the DNB (Diplôme National du Brevet) — in a few days. I decided to use this as a live test of the knowledge catalogue pipeline.

The DNB covers five written exam areas:
- **Français** — reading comprehension, grammar, dictation, extended writing
- **Mathématiques** — geometry, algebra, statistics, problem solving
- **Histoire-Géographie + EMC** — combined paper: contemporary history, geopolitics, civic education (Éducation Morale et Civique)
- **Sciences** — single paper covering SVT (life and earth sciences), Physique-Chimie, and Technologie
- **Oral / Projet** — a defended project assessed on reasoning, expression, and structured argument

For each subject, I used official French Government curriculum sources (Bulletin Officiel) to define the schema — required competencies, exam format, out-of-scope topics. Conscious of token costs on side projects, I used a pattern I rely on often: **ask Claude to produce the schema and a structured prompt, then pass that to Gemini** (via the [Gemini app](https://gemini.google.com)) to do the heavy document parsing. This keeps personal API spend manageable and plays to each model's strengths — Claude for structure and schema, Gemini for bulk content extraction.

The result is a set of curriculum markdown files, one per subject, that any parent or student can use as a revision source of truth — and that an agent can compile into a rendered GAS study app in seconds.

**Live brevet study app (no login required):**
`https://script.google.com/macros/s/AKfycbySUgHU2ynyj-GMc9_oR9qiWlwCVrNSeCwFXY_JYExxIldHYnQFfqgo_vE_uUBJg2L7/exec?nav=brevet-2026`

![Brevet study app — hub view](https://raw.githubusercontent.com/curtiskrygier/a2knowledge/master/screenshots/GAS%20Brevet/Screenshot%202026-06-21%2014.36.08.png)

![Brevet study app — Maths automatismes](https://raw.githubusercontent.com/curtiskrygier/a2knowledge/master/screenshots/GAS%20Brevet/Screenshot%202026-06-21%2014.36.49.png)

---

## Compounding Two Catalogues: The Enterprise Angle

The Brevet is a relatable example. But the same pattern applies cleanly to professional and practitioner knowledge — and the NIST AI Risk Management Framework is a good demonstration, because it's genuinely useful to the people reading this.

The **NIST AI RMF** is a U.S. Government document — public domain, formally structured, and widely referenced by AI teams navigating governance and risk. Four core functions (GOVERN, MAP, MEASURE, MANAGE), 72 subcategories, seven trustworthiness characteristics. Exactly the kind of structured, competency-mapped knowledge the pipeline was built for.

What the compound pattern looks like in practice:

1. An agent reads the **A2Knowledge catalogue** — discovers the NIST AI RMF schema, the four functions as domains, each subcategory as a required competency
2. It reads the **A2UI catalogue** — discovers the available atoms, confirms which ones are GAS-compatible, gets the invocation URL
3. It composes a payload: a reference app covering GOVERN subcategories as flashcards, MANAGE as key takeaways, with the trustworthiness characteristics as a glossary deck
4. It encodes the payload and returns a shareable GAS URL

The agent has produced a structured, curriculum-aligned AI governance reference tool. It didn't hallucinate the NIST framework — it read it from a catalogue. It didn't choose arbitrary UI components — it picked from a declared vocabulary. The result is verifiable against the source document.

This is what "declarative for agents" buys you in practice. The agent's creativity is bounded by what's in the catalogues. That's not a limitation — it's the point.

---

## A Sidebar on Media: ARD Applied to Images

The same catalogue pattern extends naturally to generated media — and the Brevet app is a concrete example.

The domain overview tab starts with a cover image. Without a media catalogue, an agent building any new study app would generate one from scratch: no registry, no reuse, no awareness that the same conceptual image was produced for someone else yesterday.

![Domain brief — no cover image yet](https://raw.githubusercontent.com/curtiskrygier/a2knowledge/master/screenshots/08_domain_brief_clean.png)

For the Brevet app, the cover was generated from a single vocab string — `"French education diploma cover, dark navy, tricolor ribbon, minimal"` — passed to Imagen 3 and stored in GCS with that string as the key.

![Domain brief — first image iteration](https://raw.githubusercontent.com/curtiskrygier/a2knowledge/master/screenshots/09_domain_brief_with_image.png)

![Domain brief — final cover image](https://raw.githubusercontent.com/curtiskrygier/a2knowledge/master/screenshots/17_image_pos.png)

The principle: **the vocabulary used to generate the image becomes its address**. An agent looking for the same concept next time presents the same vocab string — cache hit or fallback-to-generate, depending on whether the entry exists and whether the caller has write authority. Authority rules on creation (cost policy, content scope, domain rules) mean the catalogue isn't just a cache — it's a governed resource.

It's ARD applied to media. The `ai-catalog.json` entry for an image resource looks almost identical to one for an API: capability description, invocation endpoint — except the invocation is a vocab-keyed lookup. The IPTC Digital Source Type vocabulary already classifies *what kind* of asset a generated image is; a media catalogue uses that same vocabulary as the *address*.

The standard for prompt-keyed image registries doesn't exist yet. The pattern is established — DiffusionDB has 14M prompt-image pairs as a research dataset. The governed, agent-accessible catalogue layer is the natural next step.

---

## Pipeline in Practice: NIST AI RMF

Running it through the pipeline looked like this:

1. **Schema** — `schemas/pro-cert/nist/nist-ai-rmf.yaml` defining the four functions as domains, each with its categories and subcategories as required competencies, with atom hints (`glossary`, `key_takeaways`) per competency type
2. **Extraction** — scraped NIST AI 100-1 directly from `nvlpubs.nist.gov`, read the four function tables (GOVERN p.22, MAP p.26, MEASURE p.29, MANAGE p.32) and all subcategories verbatim into `nist-ai-rmf.curriculum.md` with proper section tags and competency anchors
3. **Transform** — the transformer read the curriculum glossary blocks and emitted 82 flashcards: 72 across the four functions — GOVERN (19), MAP (18), MEASURE (22), MANAGE (13) — plus the seven trustworthiness characteristics and bias categories as additional cards
4. **Render** — `payloads/nist-ai-rmf.json` → `nist_airmf_page.gs` → live GAS app (payload and renderer live in the [a2ui-catalogue](https://github.com/curtiskrygier/a2ui) repo)

**Live NIST AI RMF reference app:**
`https://script.google.com/macros/s/AKfycbySUgHU2ynyj-GMc9_oR9qiWlwCVrNSeCwFXY_JYExxIldHYnQFfqgo_vE_uUBJg2L7/exec?nav=nist-ai-rmf`

![NIST AI RMF app](https://raw.githubusercontent.com/curtiskrygier/a2knowledge/master/screenshots/NIST%20apps%20script.png)

The interesting thing about this one isn't the output — it's what the pipeline does to the source material. The PDF is 45 pages. The curriculum.md is the canonical, versioned, competency-anchored distillation of it. The GAS app is just a rendered view. If NIST updates the framework, you update the curriculum.md, re-run the transformer, redeploy. The pipeline doesn't care what the source says — it just maps sections to atoms. That separation between knowledge and rendering is what makes the compound catalogue pattern composable.

This is also the first entry in the A2Knowledge catalogue that any other agent could consume with zero IP concern. The `ai-catalog.json` entry for this resource, once the ARD registry is live, will point to a public domain knowledge base with a declared render surface. Any agent that discovers it knows: *this is structured AI governance knowledge, it renders on GAS, and you can use it freely.*

---

## Where This Is Heading

ARD is early. Most of the interesting parts — federated registry crawling, Trust Manifest verification at scale, ranked intent matching — are still ahead. But the catalog layer works *today*, and the pattern of declaring capabilities in machine-readable form and letting agents discover and compose them is real and useful right now.

My current setup:
- **A2UI catalogue** — published, 435 atoms across GAS, web, Meet stage, and Google Chat
- **A2Knowledge catalogue** — Brevet live, NIST AI RMF live, pipeline and schemas open source
- **ARD registry** — static today (my own resources), query endpoint next

The interesting shift happens when other people start publishing `ai-catalog.json` files. That's when ARD stops being a pattern you implement for yourself and starts being infrastructure that changes how agents navigate the web. The catalog layer is ready. The registry is next.

---

**Links:**
- Brevet study app: `https://script.google.com/macros/s/AKfycbySUgHU2ynyj-GMc9_oR9qiWlwCVrNSeCwFXY_JYExxIldHYnQFfqgo_vE_uUBJg2L7/exec?nav=brevet-2026`
- NIST AI RMF app: `https://script.google.com/macros/s/AKfycbySUgHU2ynyj-GMc9_oR9qiWlwCVrNSeCwFXY_JYExxIldHYnQFfqgo_vE_uUBJg2L7/exec?nav=nist-ai-rmf`
- A2UI catalogue: https://github.com/curtiskrygier/a2ui
- A2Knowledge: https://github.com/curtiskrygier/a2knowledge
- ARD registry: https://github.com/curtiskrygier/ard
