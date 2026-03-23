# Technical Specification — FDA 510(k) Review Studio v2.6  
## “Regulatory Command Center: WOW+” (Streamlit / Hugging Face Spaces)

**Deployment Target:** Hugging Face Spaces (Streamlit, single container)  
**Operational Context:** 2026, Asia/Taipei localization (English / Traditional Chinese)  
**Core APIs:** Gemini API, OpenAI API, Anthropic API, Grok API  
**Configuration:** `agents.yaml` (agent orchestration), optional `SKILL.md` (global rules)  
**Design Goal:** Preserve all v2.5 features while adding **3 new WOW AI features** plus **major visualization/observability upgrades** (interactive indicators, logs, dashboards), and ensuring a refined, user-driven “agent-by-agent” workflow with editable handoffs.

---

## 0. Change Summary (v2.6 vs v2.5)

v2.6 is a design-and-workflow upgrade that keeps every original capability intact (multi-PDF ingestion, queue selection, trimming, OCR matrix, consolidated Markdown editor, multi-agent YAML orchestration, macro-summary workflow, persistent prompting, dynamic skill execution, AI Note Keeper with AI Magics, security posture, and Hugging Face deployment). It adds:

1. **Three additional WOW AI features (new):**
   - **WOW AI Evidence Mapper** (claim → evidence traceability across files/pages)
   - **WOW AI Consistency Guardian** (cross-output contradiction & completeness checks + fix suggestions)
   - **WOW AI Regulatory Risk Radar** (section-level risk scoring with visual radar/heatmaps + mitigations)

2. **Expanded visualization and observability (new):**
   - An **Interactive Mission Control Dashboard** (system status, run queue, provider health, token/cost telemetry)
   - A **Session Event Log & Audit Trail** (redacted, exportable, filterable)
   - **Agent Run Timeline / DAG View** (editable handoff nodes, diff views, rerun from any step)
   - Rich **status indicators** across ingestion, trimming, OCR, dataset search, agent execution, and note workflows

3. **Polished “WOW UI” continuity:**
   - Maintains **Light/Dark**, **English/Traditional Chinese**, **20 painter styles + Jackpot**, “Coral reserved for critical insights,” and split-pane “Source vs Intelligence Deck”
   - Adds cohesive “Command Center” dashboards without changing core navigation concepts

---

## 1. Executive Summary

FDA 510(k) Review Studio v2.6 (“Regulatory Command Center: WOW+”) is a Streamlit-based, human-in-the-loop system for accelerating FDA-style 510(k) review and analysis. It merges high-volume document ingestion and OCR (including multimodal LLM OCR), cross-dataset regulatory context (510(k), MDR/ADR, GUDID, Recalls), and a configurable multi-agent orchestration engine driven by `agents.yaml`.

The application is designed to convert large, unstructured submission PDFs into a structured, traceable, and review-ready analytical workspace. v2.6 specifically targets the day-to-day pain points of regulatory reviewers:

- **Where did this claim come from?** → Evidence Mapper  
- **Did the summary contradict itself or omit key sections?** → Consistency Guardian  
- **What’s the risk profile across core regulatory domains?** → Regulatory Risk Radar  
- **How do I see what the system is doing and why it’s slow?** → Mission Control + telemetry + logs  
- **How can I iteratively refine agent steps without losing work?** → DAG/timeline, editable handoffs, diffs, rerun-from-step

All these enhancements are delivered without adding code in this specification and without removing any v2.5 features.

---

## 2. System Architecture Overview

### 2.1 Logical Layers (single-container, modular design)

1. **UI/UX Layer (WOW UI + dashboards)**
   - Streamlit layout (split-pane), CSS injection (glassmorphism), theme engine (light/dark), i18n engine (EN/zh-TW), painter palette engine (20 styles + Jackpot)
   - Multi-tab dashboards, interactive indicators, session log viewer, timeline/DAG visualization panels

2. **Ingestion & Queue Layer**
   - Multi-file upload + file path ingestion
   - File registry and selection table
   - Pre-processing metadata scan agent (page count, file size, PDF health)
   - Queue policies (limits, warnings, memory guardrails)

3. **Extraction & Transformation Layer**
   - Trimming engine (default first 5 pages; global/per-file ranges)
   - OCR matrix:
     - Local (PyPDF2 text + Tesseract fallback)
     - LLM OCR (Gemini multimodal models; promptable)
   - Consolidated Markdown assembler with file/page trace markers
   - Dual-view editor (Text ↔ Markdown render) with highlighting

4. **Orchestration & Intelligence Layer**
   - Agent manager (`agents.yaml` parsing + validation)
   - Step-by-step execution with editable prompts/models/tokens
   - Output editing and handoff management (output becomes next input, editable)
   - Persistent prompting + dynamic skill execution
   - New WOW AI modules (Evidence Mapper, Consistency Guardian, Risk Radar)

5. **Data & Search Layer**
   - Embedded datasets (510(k), MDR/ADR, GUDID, Recalls) loaded into DataFrames
   - rapidfuzz-powered cross-dataset fuzzy search
   - 360-degree device view aggregator (KPIs, historical risk signals)

6. **Observability & Compliance Layer (new emphasis)**
   - Telemetry counters (token estimates, latency, error rates, provider status)
   - Event log with redaction and exports
   - Run artifacts registry (inputs/outputs, versions, timestamps)

---

## 3. Configuration, Providers, and Model Support

### 3.1 Provider Abstraction

The system normalizes all LLM interactions through a provider-agnostic “execution contract”:

- **Inputs:** system prompt, user prompt, optional images, context payload, model name, max tokens, temperature, provider-specific settings
- **Outputs:** text/markdown output + metadata (token usage if available, latency, provider response id if safe)
- **Errors:** standardized error object (HTTP code class, user message, retry advice)

### 3.2 Supported Models (user-selectable per agent step)

Users can choose a model before executing **each agent**, including:

- **OpenAI:** `gpt-4o-mini`, `gpt-4.1-mini`
- **Google Gemini:** `gemini-2.5-flash`, `gemini-2.5-flash-lite`, `gemini-3-flash-preview`
- **Anthropic:** “Anthropic models” (configurable list; e.g., Claude family)
- **xAI Grok:** `grok-4-fast-reasoning`, `grok-3-mini`

**Default max_tokens:** **12,000** (user can modify per step; constrained by provider limits).  
The UI should show effective maximums and warnings if a user exceeds provider/model constraints.

### 3.3 `agents.yaml` (retained + enhanced workflow usage)

Agents remain YAML-defined with:
- agent id, name, default provider/model
- default system prompt and user prompt templates
- default temperature, max_tokens (default 12000 unless overridden)
- optional “expected output format” (Markdown, JSON-like markdown table, etc.)
- optional “guardrails” (for example: must include sections A–H)

**v2.6 adds:** per-agent “recommended dashboards” metadata (which KPIs/visualizations to show after completion).

---

## 4. Security & API Key Handling (Strict Requirements)

### 4.1 Key Sources and UI Behavior

The system supports two sources:

1. **Environment variables (preferred in Hugging Face Spaces Secrets)**
   - If a key is present in the environment:  
     - UI shows “Managed by System” badge
     - **Key input field is hidden**
     - Key value is never displayed or logged

2. **User input on webpage (only if missing in environment)**
   - Password-masked input per provider
   - Stored only in ephemeral session state
   - Not logged, not printed, not exported
   - Clear-on-purge + clear-on-session-end

### 4.2 Redaction & Safe Logging

- Event logs must never include keys, bearer tokens, or full request payloads containing secrets.
- If a provider SDK returns headers, they must be stripped before logging.
- Optional “redacted prompt logging” mode for regulated environments:
  - stores only hashes and lengths of prompts, not the raw text

### 4.3 Danger Zone (retained)

“Total Purge” deletes:
- session keys (user-entered)
- uploaded files buffers, rendered images
- OCR outputs, agent outputs, macro summaries, skill outputs
- log entries and artifacts (session-only)

Option: preserve UI preferences (theme/language/painter style) as a user choice.

---

## 5. WOW UI/UX Specification (Upgraded, preserving v2.5 design)

### 5.1 Global UI Controls (retained + refined)

- **Theme:** Light / Dark
- **Language:** English / Traditional Chinese (zh-TW)
- **Painter Style:** 20 famous painter-inspired palettes + **Jackpot** randomizer  
  - Palette applies to: accents, buttons, tabs, progress bars, KPI cards, highlight borders
  - “Coral” remains reserved as semantic highlight for critical regulatory insight

**Localization scope:** UI labels, tooltips, errors, empty-state messages, and default agent prompt templates.

### 5.2 Layout Topography (split-pane, retained)

- **Left Pane — Source Material**
  - Multi-file ingestion (upload + path list)
  - File queue table (select/exclude)
  - Trimming configuration (global + per-file)
  - OCR matrix and OCR prompt editor
  - Consolidated OCR Markdown/Text dual editor
  - Optional PDF preview panel (thumbnail or page snapshots)

- **Right Pane — Intelligence Deck**
  - Agent orchestration stepper (agent-by-agent)
  - Model/tokens/prompt overrides per step
  - Output editor per agent with “Use as next input” control
  - Macro-summary editor + persistent prompting
  - Dynamic skill execution
  - Cross-dataset search + 360-degree device view
  - **WOW AI features panel** (Evidence Mapper, Consistency Guardian, Risk Radar)
  - **WOW dashboards** (Mission Control, Timeline/DAG, Logs)

### 5.3 Dual-View Editors (retained + expanded interactions)

Each major artifact supports:
- **Text view** (editable)
- **Markdown render view** (readable, stylized)
- **Diff view (new)**: compare current vs previous version (agent output revisions, summary revisions)
- **Version snapshots (new)**: “Save checkpoint” and “Restore checkpoint” per artifact

---

## 6. Visualization, Status Indicators, and WOW Dashboards (Major Upgrade)

### 6.1 Always-Visible Status Strip (top sticky bar)

Shows real-time badges/metrics:

- **API Provider Health**
  - per provider: Key source (Env/User/None), connectivity test status (optional), last error
- **Session Workload**
  - number of PDFs ingested, selected, trimmed
  - number of pages queued for OCR
  - OCR output size (characters, approximate tokens)
- **Agent Chain**
  - current agent step index / total
  - last run status (success/warn/error)
  - “Mana” bar (retained): now computed from workload + agent complexity
- **Data Context**
  - dataset loaded counts (510(k), MDR/ADR, GUDID, Recalls)
  - last query time and results count

### 6.2 Mission Control Dashboard (new, interactive)

A dedicated dashboard tab containing:

1. **Pipeline State Machine View**
   - Ingestion → Queue → Trimming → OCR → Consolidation → Agent Steps → Macro Summary → Post-analysis (WOW AI)
   - Each node shows: status (idle/running/done/error), duration, last updated timestamp
   - Clicking a node opens details and relevant logs/artifacts

2. **Provider Telemetry Panel**
   - request count per provider (session)
   - latency charts (sparkline trend)
   - error rate and last error category
   - token usage estimate (input/output), where available
   - “Cost estimate” (optional, user-configured pricing table; clearly labeled as estimate)

3. **Resource & Memory Guardrail Indicators**
   - approximate in-memory footprint (PDF bytes + images + text buffers)
   - warnings when thresholds exceeded
   - suggested mitigations (reduce pages, choose gemini-flash-lite, etc.)

### 6.3 Session Event Log (new)

A filterable, searchable event log with:
- timestamp (Asia/Taipei)
- severity (info/warn/error)
- component (ingestion/trimming/ocr/agent/provider/ui)
- message (redacted)
- correlation id per run
- “download log” as JSON and/or Markdown report

The log should support “pin this event” and “attach to final report” for auditability.

### 6.4 Agent Run Timeline / DAG View (new)

A “workflow graph” view representing:
- each agent step as a node
- edges show handoff from prior output to next input
- nodes show model, max_tokens, duration, status, version number
- click node → open:
  - prompts used (redacted option)
  - output (text/markdown)
  - diff vs previous run
  - rerun controls from this node forward (conceptually; implementation must preserve manual edits)

This visualization is essential for human-in-the-loop traceability and supports regulated workflows.

---

## 7. Document Workflows (Retained) + Traceability Improvements

### 7.1 Multi-File Ingestion (retained)

- drag/drop multi-PDF upload (unlimited by UI; guardrails by policy)
- file path list ingestion (as allowed by environment)
- validation: file type, size, read permission, PDF parseability

### 7.2 File Queue Table (retained + improved)

Table columns:
- selected checkbox
- filename, source (upload/path)
- size, page count, last modified (if available)
- health (parseable, encrypted, corrupted)
- per-file trim override
- per-file OCR override (optional advanced mode)

Table interactions:
- select all / select none
- bulk set trim range
- bulk set OCR mode
- quick search filter

### 7.3 Trimming Engine (retained)

- default: first 5 pages
- range syntax: `1-5, 10, 15-20`
- behavior for out-of-range pages: configurable policy (clip vs warn vs block)

### 7.4 OCR Matrix (retained)

Two primary paths:

1. **Python Pack OCR**
   - local extraction with best-effort text + OCR fallback
   - fast, low cost, weaker for complex tables/scans

2. **LLM-Based OCR (Gemini multimodal)**
   - render pages to images
   - user-editable OCR prompt
   - outputs structured Markdown tables whenever possible

### 7.5 Consolidated Markdown Assembly (retained + evidence hooks)

The consolidated artifact includes strict markers:
- file name
- page number
- optional “section guess” label (e.g., “Device Description” inferred by heuristics)

**v2.6 adds “Evidence Anchors”:**
- Each marker becomes a stable anchor id used by Evidence Mapper and clickable navigation.
- Optional thumbnail cache for referenced pages (memory-guarded).

---

## 8. Agent Orchestration (Retained) — Now Fully Editable Step-by-Step

### 8.1 Agent Execution Stepper (must-have behavior)

Before running each agent, user can modify:

- provider/model (from supported list)
- system prompt and user prompt
- max_tokens (default 12000)
- temperature and other safe controls
- input payload selection:
  - consolidated OCR document
  - previous agent output
  - manual text (user paste)
  - combined (with clear ordering preview)

### 8.2 Output-as-Input Editing (explicit requirement)

After an agent runs:
- output shown in dual-view editor (text/markdown)
- user can edit output
- user clicks **“Commit as Next Input”**
- committed content becomes the next step’s input payload (with version id)
- the system stores:
  - original output snapshot
  - edited output snapshot
  - who/when (session timestamp)
  - diff summary (for traceability)

### 8.3 Macro-Summary Engine (retained)

- target: 3000–4000 words analytical report (prompt engineered)
- chunking strategy (implementation detail) is permitted but must preserve the ability for user to intervene
- summary shown in dual-view editor + diff/versioning

### 8.4 Persistent Prompting (retained)

A persistent prompt box bound to the current macro-summary state:
- can request expansions, translations, reformatting, focused gap analysis
- must preserve traceability (each persistent prompt creates a new summary version)

### 8.5 Dynamic Skill Execution (retained)

User pastes arbitrary skill description; system executes against current summary and produces a specialized report card.
**v2.6 adds:** Skill outputs can be “converted to an agent step” (optional) by saving as a node in the timeline.

---

## 9. AI Note Keeper (Retained) + AI Magics Expansion

### 9.1 Core Note Keeper Flow (retained)

- user pastes text or markdown
- system transforms into organized markdown
- user can edit in text/markdown view
- user can keep a prompt on the note (model selectable)
- keyword highlighting with user-defined colors (AI Keywords)

### 9.2 Existing AI Magics (retained baseline)

The system keeps the prior “6 AI Magics” concept (exact list may be customized, but must include):
- AI Formatting (structure into clean Markdown)
- Action items extraction
- Compliance checklist generation
- Deficiency spotting
- Summary/brief generation
- AI Keywords highlighter (user-selectable colors; Coral reserved for critical ontology)

### 9.3 Three Additional WOW AI Features (Added to Note Keeper and/or Studio)

These three new features must be accessible from the Note Keeper mode and also usable (where relevant) in the main Command Center.

---

## 10. NEW WOW AI Feature #1 — WOW AI Evidence Mapper

### 10.1 Purpose

Provide defensible traceability: map claims in summaries/agent outputs back to the original consolidated OCR anchors (file/page markers), enabling reviewers to quickly verify “where this came from.”

### 10.2 Inputs

- A target artifact: (a) macro-summary, (b) selected agent output, or (c) note
- Consolidated OCR document with file/page anchors
- Optional reviewer-configured “claim types” to detect:
  - device indications, technological characteristics, performance claims
  - test standards and results
  - software/cybersecurity assertions
  - sterilization method, shelf life
  - biocompatibility endpoints

### 10.3 Outputs

- **Evidence Table (Markdown + interactive grid)**
  - Claim snippet
  - Claim category
  - Confidence score (heuristic or model-reported)
  - Evidence anchors (file/page)
  - Extracted supporting quote(s)
  - “Potential mismatch” flag if evidence is weak/indirect

### 10.4 UX & Visualization

- Click an evidence anchor → open page snapshot or PDF view at that page (best-effort)
- A “coverage bar” showing what percentage of claims have at least one evidence anchor
- Filters by category and confidence
- Export evidence map as Markdown appendix for regulatory records

### 10.5 Guardrails

- Must clearly label uncertain mappings
- Must avoid fabricating evidence; if none found, output “No supporting anchor found”

---

## 11. NEW WOW AI Feature #2 — WOW AI Consistency Guardian

### 11.1 Purpose

Detect contradictions, omissions, and inconsistent terminology across:
- consolidated OCR vs agent outputs
- between agent outputs
- within the macro-summary itself

This supports quality and reduces “regulatory embarrassment” from internal inconsistencies.

### 11.2 Checks Performed

1. **Terminology Consistency**
   - device name variants, model numbers, product codes, units, test names
2. **Numerical Consistency**
   - conflicting values (e.g., shelf life 2 years vs 3 years)
3. **Section Completeness**
   - required sections present (configurable templates)
4. **Regulatory Red Flags**
   - “claims without evidence,” “unsupported equivalence,” “missing predicate rationale”
5. **Cross-language Stability (EN/zh-TW)**
   - if translated, ensure key technical terms remain consistent

### 11.3 Outputs

- **Issue List Dashboard**
  - severity (critical/high/medium/low)
  - affected artifact(s)
  - problematic snippet(s)
  - suggested correction text (editable)
  - “apply patch” workflow (creates a new version snapshot)

### 11.4 Visualization

- Heatmap by section (more issues = hotter)
- Trend view across revisions (issues decreasing over time)

### 11.5 Guardrails

- Must separate “verified contradictions” from “possible contradictions”
- Must keep reviewer in control: patches require explicit approval

---

## 12. NEW WOW AI Feature #3 — WOW AI Regulatory Risk Radar

### 12.1 Purpose

Produce a high-signal risk posture visualization for the submission, helping reviewers prioritize reading and follow-up questions.

### 12.2 Risk Domains (configurable template)

At minimum:
- Device description clarity
- Predicate comparison strength
- Performance testing sufficiency
- Biocompatibility completeness
- Sterilization/shelf-life packaging evidence
- Software lifecycle / cybersecurity
- Clinical evidence (if applicable)
- Labeling/IFU risks
- Post-market signals (MDR/recalls context integration)

### 12.3 Inputs

- macro-summary and/or selected agent outputs
- cross-dataset context results (MDR, recalls, GUDID attributes)
- Evidence Mapper coverage metrics (optional input signal)

### 12.4 Outputs

1. **Radar chart**: normalized 0–100 “risk attention score” per domain  
2. **Risk register table**:
   - domain, risk statement, rationale, evidence anchors, suggested mitigation questions
3. **Priority reading plan**:
   - recommended next sections to review with estimated time

### 12.5 Guardrails & Explainability

- Every risk score must include a short rationale and at least one supporting excerpt/anchor when possible.
- Must state when risk score is “low confidence” due to missing information.

---

## 13. Cross-Dataset Search & 360° Device View (Retained) + Dashboard Integration

### 13.1 Search Engine (retained)

- unified search bar
- fuzzy matching across four datasets
- results shown in interactive tables

### 13.2 360° Device View (retained + better visuals)

KPI cards:
- MDR count, recall class severity, GUDID attributes highlights
- highlight “historical risk signals” using Coral semantics
- add a small trend chart if dataset includes dates (optional)

### 13.3 Integration With Risk Radar (new)

Risk Radar should optionally incorporate:
- recall severity signals
- MDR narrative keyword flags
- GUDID attribute risk factors (implantable, MRI unsafe, single-use)

---

## 14. Observability, Error Handling, and Operational Resilience (Expanded)

### 14.1 Error Taxonomy and User Messaging

Errors categorized as:
- user correctable (missing key, invalid YAML, invalid page range)
- transient provider errors (429, timeouts)
- system constraints (memory pressure)
- data corruption (bad PDF)

Each error message must provide:
- what happened
- what to try next
- what the system did (continued/skipped/aborted)

### 14.2 Retry and Resume Semantics

- For provider timeouts: offer “retry last call”
- For agent chain: allow rerun from a step without deleting earlier artifacts
- For OCR: allow rerun OCR for a subset of files/pages

### 14.3 Audit-Ready Exports (new)

Export bundle (session-scoped, user initiated):
- consolidated OCR markdown
- macro-summary versions
- evidence map
- consistency issues report
- risk radar report
- event log (redacted)

---

## 15. Deployment Specification (Hugging Face Spaces)

### 15.1 Repository Structure (no code shown; conceptual)

Required:
- `app.py` (Streamlit entry)
- `agents.yaml` (default agents)
- `SKILL.md` (global constraints/rules; optional)
- `requirements.txt` (python dependencies)
- `packages.txt` (system dependencies for poppler/tesseract)

### 15.2 System Dependencies (retained)

- poppler-utils (PDF rendering)
- tesseract-ocr (local OCR fallback)

### 15.3 Resource Recommendations

- Minimum 16GB RAM for multi-PDF + LLM OCR workflows
- Guidance in UI when memory thresholds approached
- Optional “low-resource mode” toggle:
  - reduced image resolution
  - default to flash-lite models
  - aggressive trimming defaults

---

## 16. Acceptance Criteria (v2.6)

1. WOW UI supports Light/Dark, EN/zh-TW, 20 painter styles + Jackpot without losing editor content during rerenders.
2. Environment API keys are never displayed; input fields appear only when missing in environment.
3. Multi-file ingestion supports upload + path list; file queue selection controls downstream processing.
4. Trimming defaults to first 5 pages; supports custom ranges and per-file overrides with clear out-of-range behavior.
5. OCR matrix supports Python Pack and LLM OCR with user-editable OCR prompts; output consolidated with file/page anchors.
6. Dual-view editors support text/markdown render plus new diff and version checkpointing.
7. Agent execution is step-by-step; user can modify model/prompt/max_tokens (default 12000) before each run.
8. User can edit an agent output and commit it as the next agent input; timeline/DAG reflects the handoff.
9. Macro-summary targets 3000–4000 words and supports persistent prompting with version history.
10. Dynamic skill execution works as specified and can be saved as a timeline node (optional).
11. AI Note Keeper supports paste-to-organized-markdown, model-selectable prompting, and AI Magics including keyword coloring.
12. New dashboards exist: Mission Control, Session Event Log, Agent Timeline/DAG.
13. WOW AI Evidence Mapper produces claim-to-anchor mapping with exportable evidence table.
14. WOW AI Consistency Guardian flags contradictions/omissions and supports reviewer-approved patches.
15. WOW AI Regulatory Risk Radar generates radar visualization + risk register + priority reading plan.
16. Cross-dataset search remains fast and integrated with device view and risk radar.
17. All logs are redacted and exportable; no secrets stored.
18. Danger Zone purge clears sensitive artifacts and session keys; optional preservation of UI preferences.
19. Provider failures do not crash the app; system provides actionable recovery controls.
20. The entire system deploys on Hugging Face Spaces within a single container.

---

# Appendix — 20 Comprehensive Follow-up Questions

1. **Evidence standard:** For the Evidence Mapper, should “evidence” require a direct quote match, or is topical proximity within the same page acceptable when exact phrasing differs?
2. **Anchor granularity:** Do you want anchors at **page-level only**, or also **block-level** (e.g., detected headings/tables) to improve precision?
3. **Reviewer workflow:** When Evidence Mapper finds weak support, should it automatically draft a “request for clarification” question list?
4. **Consistency patching:** Should Consistency Guardian patches be applied as inline edits to the summary, or as tracked “suggestions” that require manual acceptance line-by-line?
5. **Regulatory templates:** Which section template(s) should completeness checks use (e.g., FDA Refuse-to-Accept-like checklist, internal SOP template, device class-specific templates)?
6. **Risk scoring policy:** Should Risk Radar scores be purely rule-based, LLM-derived, or a hybrid? If hybrid, which domains must remain rule-based for defensibility?
7. **Risk calibration:** Do you want different Radar weighting profiles per product code/device category (e.g., software-heavy vs implantable)?
8. **Cross-dataset integration:** Should MDR/recall/GUDID signals automatically elevate risk scores, or remain “context only” unless the reviewer explicitly enables it?
9. **Model governance:** Do you require an allowlist per workspace (e.g., disable Grok in some environments) and per-feature restrictions (e.g., OCR only allowed on Gemini)?
10. **Token/cost telemetry:** Should the dashboard show estimated cost by default, or require explicit opt-in due to sensitivity and variability in pricing?
11. **Latency strategy:** For LLM OCR on many pages, should we prioritize parallelism (faster but higher rate-limit risk) or sequential batching (slower but safer)?
12. **Max upload limits:** What hard limits should be enforced on number of PDFs, total MB, and total rendered pages to protect Hugging Face memory constraints?
13. **PDF preview needs:** Is an embedded PDF viewer with page jumping mandatory in v2.6, or is a page snapshot viewer sufficient initially?
14. **Localization depth:** Should agent prompts also auto-switch language (EN/zh-TW) by default, or only the UI labels—leaving prompts unchanged unless user selects “translate prompts”?
15. **Terminology dictionary:** Do you have an internal controlled vocabulary (device names, abbreviations, test standard naming) that Consistency Guardian should enforce?
16. **Note Keeper governance:** Should Note Keeper artifacts be exportable separately from Command Center artifacts, and should it support multiple named notes per session?
17. **Keyword coloring precedence:** For overlapping keywords with different colors, should priority be longest-match-first, user-order-first, or severity-based?
18. **Audit requirements:** Do you need a “regulatory audit pack” export that includes a fixed structure and metadata headers (reviewer, date, models used, dataset versions)?
19. **Privacy controls:** Should there be an optional pre-flight redaction step before sending any text/images to external APIs (even if it reduces accuracy)?
20. **Multi-user behavior:** On Hugging Face Spaces, do you want any explicit concurrency controls (queueing agent runs per session) to prevent resource contention and timeouts?

---
