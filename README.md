<img width="1024" height="304" alt="Tenderstack" src="banner.png" />

# Tenderstack

A multi-agent framework for producing tender, RFP, and ITT responses from a firm's own bid library. Thirteen specialist agents across five teams coordinate to ingest historic bids, research buyers and competitors, draft submission-ready responses, and learn from every outcome. All of it runs inside an agentic coding CLI (Claude Code, Codex, or similar) rather than as a standalone SaaS product.

> This repository is a public overview of the project. The source code is maintained privately.

---

## Table of Contents

- [How It Works](#how-it-works)
- [Agent Teams](#agent-teams)
  - [Bid Team](#bid-team)
  - [Research Team](#research-team)
  - [Commercial Operations](#commercial-operations)
  - [Feedback Team](#feedback-team)
  - [Knowledge Curator (Cross-Cutting)](#knowledge-curator-cross-cutting)
- [Pipelines](#pipelines)
  - [Pipeline 1: Bid Response](#pipeline-1-bid-response)
  - [Pipeline 2: Feedback Loop](#pipeline-2-feedback-loop)
- [Project Memory Wiki](#project-memory-wiki)
- [Architecture](#architecture)
  - [Runtime Layer](#runtime-layer)
  - [Database Schema](#database-schema)
  - [Obsidian Vault & Wiki](#obsidian-vault--wiki)
- [Repository Layout](#repository-layout)
- [Requirements](#requirements)
- [Installation & First Use](#installation--first-use)
- [Use Cases](#use-cases)
- [Status & Roadmap](#status--roadmap)
- [Compliance](#compliance)
- [License](#license)

---

## How It Works

You bring your historic bid library (won and lost responses, case studies, methodology documents, pricing sheets) and a live tender to respond to. The framework's agents, coordinated by `AGENTS.md` and run through your agentic CLI, handle the rest:

1. **Ingest** your library into a tagged, linked Obsidian knowledge base
2. **Parse** the live tender into structured requirements, gap-checked against your library
3. **Research** the buyer and named competitors with real web search, every finding source-cited
4. **Design** a technical solution from site surveys, equipment inventories, and infrastructure evidence
5. **Price** the bid from technical scope, staffing availability, and commercial constraints
6. **Draft** win themes, an executive summary, and submission-ready section responses, with every claim evidence-backed
7. **Check** coverage against the buyer's document inventory before export
8. **Export** to `.docx` for human review
9. **Learn**: when the result comes in, run the feedback pipeline, update win-theme performance counters, and synthesize cross-bid wiki pages so the next bid starts stronger

Every agent calls the mechanical runtime layer (`runtime/cli.py`) rather than writing state directly. Thirteen independently-prompted agents share one database and one filesystem without corrupting each other's work.

---

## Agent Teams

### Bid Team

The core production pipeline. It ingests your library and produces submission-ready proposals.

| Agent | Role | Trigger |
|-------|------|---------|
| **document-controller** | Ingests historic bid library and finished responses. Chunks content, tags against a controlled taxonomy, writes Obsidian-style vault files with `[[wikilinks]]`, and runs a mandatory content checklist and coverage self-check before declaring a document done. | On-demand, per document |
| **rfp-analyst** | Parses a live RFP/ITT into structured requirements (technical, commercial, compliance, evaluation), categorises each as mandatory or not, and gap-checks against the vault. Every requirement tagged `covered`, `missing`, or `needs_review`. | On-demand, per live tender |
| **bid-manager** | Makes the formal bid/no-bid call with rationale, sets submission deadlines, writes C-suite briefs and ops overviews, and produces the final `submission_readiness` coverage check, cross-referencing drafted sections against the buyer's document inventory and flagging every `[NEEDS INPUT]` placeholder. | After RFP analysis (step 3a) and after drafting (step 9) |
| **bid-writer** | Strategic proposal architect. Develops 3-5 win themes per bid, writes the executive summary (the proposal's closing argument, placed first), designs the three-act narrative architecture, and drafts submission-ready `section_response` prose, with every covered requirement answered from its backing vault chunk, every gap flagged, and every word-count limit hit. Pulls context from vault, research, technical design, pricing, and the project memory wiki. | After research, technical design, and pricing are complete |
| **technical-solutions-architect** | Analyses site specs, equipment inventories, infrastructure layouts, and RAG documents to design a proposed technical solution. Writes structured `site_assets` and a `technical_solutions` summary addressing specific requirement IDs. Evidence-grounded, not invented from the RFP text alone. | On-demand, when site/infrastructure documents exist for the bid |

### Research Team

Real web research, source-cited. Feeds the Bid Team with evidence rather than sector stereotypes.

| Agent | Role | Trigger |
|-------|------|---------|
| **buyer-researcher** | Deep dive on a specific buyer: strategic priorities, sustainability commitments, core values, recent news. Every finding has a `source_url`. Writes both structured DB rows and a browsable vault dossier. | On-demand, per live bid |
| **competitor-researcher** | Researches named competitors: strengths, weaknesses, pricing signals, case studies. Findings are sector-scoped and reusable across bids (same pattern as `win_themes`). Every finding source-cited. | On-demand, per live bid or sector sweep |

### Commercial Operations

Turns site maps, staffing data, and technical scope into a priced offering.

| Agent | Role | Trigger |
|-------|------|---------|
| **chief-commercial-officer** | Collates technical solution evidence, staffing availability, and commercial requirements into an initial price offering. The one agent in this framework that spawns another agent. It delegates regional staffing surveys to `staffing-analyst` when a site map exists. | After technical solution is designed |
| **staffing-analyst** | Reads site maps and staff numbers to produce regional availability reports: headcount by region with role breakdowns. Feeds the CCO's pricing calculus. | Spawned by CCO |

### Feedback Team

No lead role. Fixed Ingester → Verifier → Overseer sequence, run after every bid result.

| Agent | Role | Trigger |
|-------|------|---------|
| **ingester** | Stores the evaluator's feedback document and sets the bid outcome (`won`/`lost`) on the `bids` row. | First in feedback pipeline |
| **verifier** | Systematically checks every evaluator marker against what was actually submitted. Writes `verification_findings`: `missed`, `weak`, `miscited`, `contradicted`, or `strong`. | Immediately after ingester |
| **overseer** | Updates win-theme performance counters (`times_won`, `win_rate`) and writes qualitative `pattern_notes`, specific and checkable observations from Verifier's findings. Never edits theme wording. | Immediately after verifier |

### Knowledge Curator (Cross-Cutting)

The framework's memory. Sits outside any single team: reads from everyone, writes to no one's tables.

| Agent | Role | Trigger |
|-------|------|---------|
| **knowledge-curator** | Synthesizes cross-bid wiki pages from existing structured data: sector playbooks, buyer profiles, competitor dossiers, decision post-mortems, methodology patterns. Second-order knowledge only: connects what other agents already wrote, never invents new facts, never spawns other agents, never does primary research. Every page has `[[wikilinks]]` and is designed to be opened in Obsidian. | Post-Pipeline-2 (automatic), on-demand, or weekly CI |

---

## Pipelines

### Pipeline 1: Bid Response

On-demand. You orchestrate: spawn each specialist as a real subagent, in order. Steps 3a/4/5 can run in parallel once step 3 completes.

```
1. open-bid (you)                     Create the bids row
2. document-controller                Ingest library documents → vault
3. rfp-analyst                        Parse RFP → requirements + gap-check
   3a. bid-manager                    Bid/no-bid decision (stop here if no_bid)
4. technical-solutions-architect      Design solution from site evidence
5. buyer-researcher + competitor-researcher (parallel)
6. chief-commercial-officer           Price the bid (spawns staffing-analyst)
7. Build section inventory (you)      Every document the buyer requires
8. bid-writer                         Draft win themes → exec summary → section responses
9. bid-manager (coverage check)       submission_readiness brief
10. export-docx (you)                 Export to Word
11. knowledge-curator (optional)      Update buyer/competitor wiki profiles
```

### Pipeline 2: Feedback Loop

Triggered when a bid result and feedback document are available. Fixed sequence, no lead role. Ingester → Verifier → Overseer → Knowledge Curator.

```
1. ingester          Store feedback doc + set outcome
2. verifier          Check evaluator comments against submission
3. overseer          Update win-theme counters + write pattern notes
4. knowledge-curator Update wiki playbooks, buyer profiles, decision post-mortems
```

Pipeline 2 is the self-improvement loop. Pattern notes written here become playbook content; playbook content feeds bid-writer on the next Pipeline 1 run.

---

## Project Memory Wiki

New in this release. A cross-bid knowledge layer that learns from every outcome so the next bid starts with richer context.

```
wiki/
  decisions/       Bid post-mortems: decision, rationale, outcome, lessons
  competitors/     Per-competitor profiles: strengths, weaknesses, pricing signals
  buyers/          Per-buyer profiles: priorities, values, sustainability, bid history
  playbooks/       Per-sector win patterns: what works, what evaluators penalise
  methodology/     Pricing models, technical patterns, staffing ratios across bids
  other/           Catch-all for new categories
```

**Every page is Obsidian markdown**: YAML frontmatter plus body with `[[wikilinks]]`. Open `wiki/` as an Obsidian vault for graph view, backlinks, and visual knowledge navigation.

**How it works:**
- `knowledge-curator` reads DB tables that other agents wrote: `pattern_notes`, `buyer_research`, `competitor_research`, `bid_decisions`, `verification_findings`, `technical_solutions`, `price_offerings`, `staffing_availability`
- Synthesizes readable wiki pages (never invents; "insufficient data" is a valid section)
- Every page includes a `## See Also` footer linking to related pages and vault chunks
- `bid-writer` automatically receives matching wiki pages in its context block via `get-bid-writer-context`
- Wiki refreshes after every Pipeline 2 completion, plus weekly CI for web-dependent topics

**The playbook is the highest-value artifact.** After a few bids in the same sector, `wiki/playbooks/edtech.md` contains concrete, traceable guidance: which win themes have the highest win rate, which evidence types evaluators reward, which gaps recur, and a "before you draft" checklist for bid-writer.

See [`knowledge_base.md`](knowledge_base.md) for linking conventions and full topic documentation.

---

## Architecture

### Runtime Layer

`runtime/cli.py` is the only entry point to the database and filesystem. Every agent calls it via Bash. Never raw SQL, never hand-written frontmatter. Every subcommand prints JSON to stdout and exits non-zero with a message on stderr on failure.

```
python runtime/cli.py <subcommand> --help    # lists arguments for any subcommand
```

**Modules:**
- `runtime/db.py`: mechanical database functions. No judgment lives here.
- `runtime/vault.py`: chunk file write/read/find with frontmatter parsing.
- `runtime/wiki.py`: wiki page write/read/list/find. Mirrors vault.py.
- `runtime/extract.py`: `.docx` / `.pdf` → plain text extraction.
- `runtime/docx_export.py`: markdown → `.docx` for human review (headings, bold, bullets, tables).

### Database Schema

17 SQLite tables, each owned by a specific agent:

| Table | Owned By | Purpose |
|-------|----------|---------|
| `bids` | Orchestrator | One row per bid, the root every other table joins from |
| `canonical_tags` | document-controller | Controlled sector/service_line taxonomy |
| `win_themes` | bid-writer / overseer | Reusable win themes with win/loss counters |
| `bid_content_log` | bid-writer | Every drafted deliverable (win themes, exec summary, narrative, section responses) |
| `requirements` | rfp-analyst | Structured RFP requirements with gap status |
| `bid_decisions` | bid-manager | Formal bid/no-bid calls with rationale |
| `bid_manager_briefs` | bid-manager | Ops overviews, deadline summaries, C-suite briefs, submission readiness |
| `site_assets` | technical-solutions-architect | Structured facts about a site's existing technical estate |
| `technical_solutions` | technical-solutions-architect | Proposed solution designs |
| `staffing_availability` | staffing-analyst | Regional headcount by role |
| `price_offerings` | chief-commercial-officer | Collated initial price offerings |
| `buyer_research` | buyer-researcher | Per-bid buyer findings with source URLs |
| `competitor_research` | competitor-researcher | Sector-wide competitor intelligence |
| `feedback_documents` | ingester | Raw evaluator feedback |
| `verification_findings` | verifier | Marker-by-marker submission check |
| `pattern_notes` | overseer | Qualitative feedback patterns |
| `wiki_pages` | knowledge-curator | Index over wiki/ markdown files with staleness hashes |

### Obsidian Vault & Wiki

Two complementary markdown directories:

- **`vault/`**: per-bid library content. Chunks tagged by sector/service_line, linked with `[[wikilinks]]`. Built by `document-controller`. Read by every Bid Team agent. Open in Obsidian to browse your firm's bid library as a knowledge graph.

- **`wiki/`**: cross-bid synthesized knowledge. Sector playbooks, buyer profiles, competitor dossiers, decision post-mortems. Built by `knowledge-curator`. Read by `bid-writer` (automatically via context) and by humans browsing in Obsidian.

Same format: YAML frontmatter, markdown body, and `[[wikilinks]]`. Same Obsidian compatibility. Different purpose. Vault is what's in our library. Wiki is what we've learned across bids.

---

## Repository Layout

```
tenderstack/
├── agents/                          Tool-agnostic persona definitions (source of truth)
│   ├── bid_team/                    document-controller, rfp-analyst, bid-writer,
│   │                                  bid-manager, technical-solutions-architect,
│   │                                  orchestrator (stub)
│   ├── research_team/               buyer-researcher, competitor-researcher
│   ├── commercial_operations_team/  chief-commercial-officer, staffing-analyst
│   ├── feedback_team/               ingester, verifier, overseer
│   └── cross_cutting/               knowledge-curator
├── .claude/agents/                  Claude Code subagent configs (13 agents)
├── .codex/agents/                   Codex subagent configs (13 agents)
├── runtime/                         Mechanical layer, the only code that touches state
│   ├── cli.py                       JSON-over-stdout CLI, ~55 subcommands
│   ├── db.py                        Database functions (no judgment)
│   ├── vault.py                     Chunk file write/read/find
│   ├── wiki.py                      Wiki page write/read/list/find
│   ├── extract.py                   .docx/.pdf → plain text
│   ├── docx_export.py               Markdown → .docx
│   └── requirements.txt             Python dependencies
├── db/
│   ├── schema.sql                   17 tables, indexes, CHECK constraints
│   ├── config.yaml                  vault/db path configuration
│   └── init_db.py                   Safe init (refuses to overwrite existing DB)
├── vault/                           Obsidian markdown knowledge base
├── wiki/                            Cross-bid memory wiki (6 topics)
│   ├── decisions/                   Bid post-mortems
│   ├── competitors/                 Competitor profiles
│   ├── buyers/                      Buyer profiles
│   ├── playbooks/                   Sector win/loss playbooks
│   ├── methodology/                 Pricing, technical, staffing patterns
│   └── other/                       Catch-all
├── .github/workflows/
│   └── wiki-refresh.yml             Weekly CI wiki staleness check
├── AGENTS.md                        Pipeline orchestration (tool-agnostic)
├── CLAUDE.md                        Claude Code wrapper (imports AGENTS.md)
├── README.md                        This file
├── blueprint.md                     Original full design vision
├── setup.md                         First-run onboarding wizard
├── knowledge_base.md                Wiki documentation + linking conventions
├── compliance.md                    UK compliance guidance (placeholder)
└── ROADMAP.md                       Prioritised feature backlog
```

---

## Requirements

- **Python 3.10+**
- **An agentic coding CLI**, built and tested against Claude Code and Codex. Other tools work but may lack real subagent spawning.
- **Web search/fetch access**, required for `buyer-researcher` and `competitor-researcher`. Every other agent works offline against your own documents and database.
- **Python packages:** `pyyaml`, `python-docx`, `pypdf`

---

## Installation & First Use

The source for this framework isn't published in this repository. If you'd like to see it running, talk about access, or discuss licensing, get in touch directly.

---

## Use Cases

### Ingest a Historic Bid Library

> "I have 4 won bid responses and 2 lost ones. Ingest them into the library."

`document-controller` chunks each document by meaning (case studies stay whole, methodology sections stay whole), tags them against the controlled taxonomy, writes Obsidian vault files with `[[wikilinks]]`, and runs a coverage self-check: leadership teams, accreditations, case studies, pricing models all accounted for.

### Respond to a Live Tender

> "Here's the ITT from Example Academy Trust. We're bidding for their ICT Managed Service. Deadline is September 1st."

The full Pipeline 1 runs: RFP analysis, bid/no-bid call, buyer/competitor research, technical solution design, pricing, win theme development, executive summary, section responses, coverage check, Word export. `bid-writer` gets the buyer's own language from `buyer-researcher`, the competitor's likely positioning from `competitor-researcher`, and the sector playbook from the wiki, so the draft is grounded in evidence, not generic proposal boilerplate.

### Learn From a Result

> "We lost the Example bid. Here's the evaluator's feedback document."

Pipeline 2 runs: `ingester` stores the feedback, `verifier` checks every marker against what was submitted, `overseer` updates win-theme counters and writes pattern notes, `knowledge-curator` updates the sector playbook, so the next bid in that sector starts with "evaluators consistently penalise missing named contacts" already documented.

### Browse the Knowledge Graph

Open `vault/` and `wiki/` as an Obsidian vault. The graph view shows your entire bid library, every buyer profile, every competitor dossier, and how they connect: chunks linked by sector, wiki pages linked to vault evidence, win themes linked to outcomes.

---

## Status & Roadmap

**Built and runtime-wired:**
- All 13 agents across 5 teams
- Both pipelines (Bid Response + Feedback Loop)
- Project memory wiki with Obsidian `[[wikilinks]]`
- `.claude/agents/` configs for Claude Code
- `.codex/agents/` configs for Codex
- Document extraction for `.docx` / `.pdf` source files
- Word export for human review
- Weekly CI wiki staleness check

**Known gaps:**
- Orchestrator remains a stub; `AGENTS.md` fills the role
- No persistent staffing map for the firm's own operational footprint
- No real pricing methodology table (CCO prices from vault precedent)
- No document revision mechanism (`superseded_by` on `bid_content_log`)
- No dedicated action/audit log
- Compliance module not yet written

**Built, not yet proven against real volume:** the wiki layer is designed to handle dozens of bids per sector, but the synthesis quality depends on having enough data to spot real patterns. A playbook built from 2 bids is honest about its sample size; a playbook from 20 bids is the framework's real long-term advantage.

---

## Compliance

UK-specific guidance on ISO regulation, data protection, and data handling is planned but not yet built out. No agent enforces compliance rules yet; `bid-writer` drafts submission content without a dedicated compliance review pass.

---

## License

This project is not currently open source. All rights reserved. Get in touch if you want to talk about licensing or access.
