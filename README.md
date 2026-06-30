# Agent Skills — Comprehensive Reference Card

> **Platform**: Claude.ai · Claude Code · Cowork  
> **Version context**: Skills system as of Claude Sonnet 4 / May 2026  
> **Purpose**: Author, deploy, test, iterate, package, and optimize Agent Skills

---

## Table of Contents

1. [What Are Agent Skills?](#1-what-are-agent-skills)
2. [Anatomy of a Skill](#2-anatomy-of-a-skill)
3. [SKILL.md — Full Specification](#3-skillmd--full-specification)
4. [Progressive Disclosure Loading Model](#4-progressive-disclosure-loading-model)
5. [Bundled Resource Types](#5-bundled-resource-types)
6. [How Triggering Works](#6-how-triggering-works)
7. [Writing Patterns & Style Guide](#7-writing-patterns--style-guide)
8. [Directory Organisation Patterns](#8-directory-organisation-patterns)
9. [Eval & Testing System](#9-eval--testing-system)
10. [JSON Schemas — All Types](#10-json-schemas--all-types)
11. [CLI Scripts Reference](#11-cli-scripts-reference)
12. [Description Optimisation (run_loop)](#12-description-optimisation-run_loop)
13. [Platform Differences Table](#13-platform-differences-table)
14. [Design Patterns](#14-design-patterns)
15. [Best Practices Checklist](#15-best-practices-checklist)
16. [Anti-Patterns to Avoid](#16-anti-patterns-to-avoid)
17. [End-to-End Workflow Cheat Sheet](#17-end-to-end-workflow-cheat-sheet)
18. [Full Example: Minimal Skill](#18-full-example-minimal-skill)
19. [Full Example: Multi-Domain Skill with Scripts](#19-full-example-multi-domain-skill-with-scripts)

---

## 1. What Are Agent Skills?

Agent Skills are **markdown-based instruction packages** that extend Claude with domain-specific knowledge, reusable scripts, and reference materials. They act as a lightweight plugin/prompt-injection layer: Claude reads the skill's `SKILL.md` and follows its instructions for the matched task, optionally executing bundled scripts or reading bundled reference files.

| Concept | Analogy |
|---|---|
| `SKILL.md` | System prompt injected at task time |
| `scripts/` | Helper executables bundled with the skill |
| `references/` | On-demand documentation loaded into context |
| `assets/` | Templates, fonts, icons used in output |
| `.skill` file | Distributable package (ZIP variant) |
| `evals/` | Automated test suite for the skill |

Skills are **not code plugins** — they are instructions. Claude reads them and executes accordingly, delegating deterministic or repetitive work to bundled scripts when available.

---

## 2. Anatomy of a Skill

```
my-skill/
├── SKILL.md                  ← Required. YAML frontmatter + instruction body.
├── evals/
│   ├── evals.json            ← Test prompts + assertions
│   └── files/                ← Sample input files for evals
├── scripts/
│   └── process.py            ← Bundled helper scripts (optional)
├── references/
│   ├── README.md             ← Table of contents for large ref libraries
│   ├── aws.md                ← Domain-variant reference docs
│   └── gcp.md
└── assets/
    ├── template.docx         ← Output templates
    └── logo.png              ← Branding / icons
```

**Package output** (after `package_skill.py`):

```
my-skill.skill                ← Distributable; user installs via Claude.ai settings
```

---

## 3. SKILL.md — Full Specification

### 3.1 YAML Frontmatter

```yaml
---
name: my-skill                         # Required. Identifier. Must match directory name.
description: |                         # Required. Primary triggering mechanism.
  Short imperative description of what the skill does.
  Include when to use it, specific contexts, keywords, and
  edge cases. Be "a little pushy" — list every scenario
  where this skill should fire.
license: Complete terms in LICENSE.txt  # Optional. Path to license file.
compatibility:                          # Optional. Rarely needed.
  tools: [bash, python]
  min_context: 8000
---
```

### 3.2 Frontmatter Field Reference

| Field | Required | Purpose | Notes |
|---|---|---|---|
| `name` | ✅ | Skill identifier | Must match directory name; used in packaging |
| `description` | ✅ | Triggering + summary | Entire trigger logic lives here, not in body |
| `license` | ❌ | License pointer | Path relative to skill root |
| `compatibility` | ❌ | Tool/env requirements | Use sparingly; only hard blockers |

### 3.3 Body Structure

The body is standard Markdown. Sections commonly used:

```markdown
# Skill Name

## Overview
One-paragraph summary of what the skill enables.

## Workflow / Process
Numbered steps — imperative voice.

## Output Format / Template
Exact structure Claude must follow when producing output.

## Examples
Input/output pairs demonstrating expected behaviour.

## Reference Files
When and why to read each file in references/.

## Edge Cases
Explicit handling for known tricky inputs.
```

**Ideal length**: under 500 lines. If approaching 500, add a `references/` subdocument and point to it with guidance on when to load it.

---

## 4. Progressive Disclosure Loading Model

Skills use a **three-level hierarchy** to manage context budget:

```
Level 1  ─── Always in context (~100 words)
             name + description (frontmatter only)
             ↓ Skill triggers
Level 2  ─── In context when skill is active (<500 lines)
             Full SKILL.md body
             ↓ Needs deeper info
Level 3  ─── Loaded on demand (unlimited)
             references/, assets/ (read by Claude)
             scripts/ (executed without reading)
```

**Why this matters**: Claude sees Level 1 for *every* conversation. Level 2 only loads when the skill triggers. Level 3 loads only when SKILL.md explicitly directs Claude to read a reference or run a script. This keeps context usage efficient across millions of invocations.

| Level | When loaded | Content | Token impact |
|---|---|---|---|
| 1 — Metadata | Always | `name` + `description` | ~100 tokens |
| 2 — Body | On trigger | Full SKILL.md | ~1,000–4,000 tokens |
| 3 — Resources | On demand | References, assets, scripts | Variable / zero for scripts |

---

## 5. Bundled Resource Types

### 5.1 `scripts/` — Executable Helpers

Scripts handle work that is **deterministic, repetitive, or environment-specific** (e.g., generating a DOCX from a template, running a PDF merger, producing a chart). They:

- Are executed via `bash` / `python` without being read into context
- Encode hard-won trial-and-error (library calls, rendering quirks)
- Save every future skill invocation from reinventing the wheel
- Should be written when the same multi-step pattern appears across ≥ 2 test runs

```python
# scripts/generate_report.py  ← typical pattern
import sys, json
from pathlib import Path
# ... deterministic logic
```

Call pattern in SKILL.md:
```markdown
Run the generation script:
python scripts/generate_report.py --input {input_path} --output {output_path}
```

### 5.2 `references/` — On-Demand Docs

Long reference material (API specs, brand guidelines, legal text, framework docs) that would bloat the skill body if inlined. SKILL.md should specify *when* to read each file:

```markdown
## Reference Files
- Read `references/aws.md` only for AWS deployment targets
- Read `references/gcp.md` only for GCP deployment targets
- For large files (>300 lines), the file includes a ToC — read the ToC first
```

### 5.3 `assets/` — Static Files for Output

Templates, fonts, icons, SVGs that Claude should copy or reference when producing output. Reference them explicitly in SKILL.md.

---

## 6. How Triggering Works

Claude sees **all skill names + descriptions** in its `available_skills` context block. It decides whether to read a skill's body based on that description alone.

**Key insight**: Claude only consults skills for tasks it **cannot easily handle on its own**. Simple one-step queries ("read this PDF") may not trigger a skill even if the description matches, because Claude handles them natively. Complex, multi-step, or specialised queries reliably trigger skills.

### 6.1 Description Writing Rules

| Rule | Detail |
|---|---|
| **Include what it does** | Clear verb-first summary |
| **Include when to trigger** | List specific contexts, file types, user phrases |
| **Include edge cases** | Adjacent domains where this skill should win |
| **Be "pushy"** | Err toward over-stating coverage to combat under-triggering |
| **No body content** | All "when to use" logic goes in the description, not the body |
| **Use keywords** | Include synonyms users might type casually or formally |

**Under-trigger prevention example**:

```yaml
# ❌ Too passive
description: Helps create Excel spreadsheets.

# ✅ Pushier, more coverage
description: >
  Create, edit, read, fix, and transform Excel spreadsheets and CSV files.
  Use when the user mentions .xlsx, .xlsm, .csv, tabular data, pivot tables,
  formulas, charts in spreadsheets, data cleaning, or any deliverable that
  should be a spreadsheet. Also trigger for financial models, trackers,
  dashboards in Excel, and requests to 'add a column' or 'compute X in my file'.
```

---

## 7. Writing Patterns & Style Guide

### 7.1 Voice & Tone

- Use **imperative form**: "Read the file", "Generate the report", "Always include headers"
- Prefer explaining **why** over issuing rigid MUSTs — LLMs reason better with context
- Avoid oppressive `ALWAYS`/`NEVER` in caps when a rationale will do
- Keep instructions **general**, not over-fitted to narrow examples

```markdown
# ❌ Rigid, over-specific
ALWAYS output exactly 3 bullet points per section, never more, never less.

# ✅ Reasoned, general
Group findings into concise sections. Aim for 2–4 bullet points per section —
enough to be informative without padding. The goal is scannability, not length.
```

### 7.2 Defining Output Formats

```markdown
## Report Structure
Use this exact template for every output:

# [Title]
## Executive Summary
## Key Findings
## Recommendations
## Next Steps
```

### 7.3 Examples Pattern

```markdown
## Examples

**Example 1 — Invoice extraction**
Input: "Pull all amounts from invoice.pdf"
Output: JSON array with keys: `line_item`, `amount`, `currency`, `date`

**Example 2 — Multi-page merge**
Input: "Combine report_jan.pdf and report_feb.pdf"
Output: Single PDF with bookmarks; file saved to /mnt/user-data/outputs/
```

### 7.4 Reference File Pointers

```markdown
## Reference Files
This skill ships with three reference documents:
- `references/api_spec.md` — Full REST API reference. Load when the user
  asks about a specific endpoint or parameter not covered by the overview.
- `references/error_codes.md` — All known error codes and recovery steps.
  Load when troubleshooting a failing request.
- `references/changelog.md` — Version history. Load only when the user asks
  about version differences.
```

---

## 8. Directory Organisation Patterns

### 8.1 Single-Domain Skill

```
image-resizer/
├── SKILL.md
└── scripts/resize.py
```

### 8.2 Multi-Domain Skill (Variant Fan-Out)

```
cloud-deploy/
├── SKILL.md          ← Workflow + domain selection logic
└── references/
    ├── aws.md
    ├── gcp.md
    └── azure.md
```

SKILL.md instructs Claude to read only the relevant variant. Prevents loading unnecessary context for every invocation.

### 8.3 Large Reference Skill

```
tax-advisor/
├── SKILL.md
└── references/
    ├── README.md        ← ToC for the library
    ├── us_federal.md
    ├── eu_vat.md
    └── uk_hmrc.md
```

For any reference file > 300 lines, include a table of contents at the top so Claude can locate sections without reading the full file.

### 8.4 Script-Heavy Skill

```
docx-generator/
├── SKILL.md
├── scripts/
│   ├── generate.py       ← Main generation
│   ├── validate.py       ← Post-generation checks
│   └── requirements.txt
└── assets/
    └── template.docx
```

---

## 9. Eval & Testing System

### 9.1 Eval Lifecycle

```
1. Write evals.json          ← 2–3 realistic prompts + expected_output
         ↓
2. Spawn with-skill runs      ← Subagents follow the skill's instructions
   Spawn baseline runs        ← Same prompts, no skill (or old skill)
         ↓
3. Draft assertions           ← While runs are in progress
         ↓
4. Grade runs                 ← Grader agent evaluates assertions → grading.json
         ↓
5. Aggregate benchmark        ← aggregate_benchmark.py → benchmark.json
         ↓
6. Launch eval viewer         ← generate_review.py → browser HTML
         ↓
7. Human reviews outputs      ← Qualitative feedback + quantitative scores
         ↓
8. Improve skill              ← Update SKILL.md, bump iteration
         ↓
9. Repeat from step 2
```

### 9.2 Workspace Layout

```
my-skill-workspace/
├── iteration-1/
│   ├── eval-0-invoice-extraction/
│   │   ├── with_skill/
│   │   │   ├── outputs/         ← Files produced by the skill run
│   │   │   │   └── metrics.json
│   │   │   ├── grading.json
│   │   │   └── timing.json
│   │   ├── without_skill/
│   │   │   └── ...              ← Same structure, baseline run
│   │   └── eval_metadata.json
│   ├── benchmark.json
│   └── benchmark.md
└── iteration-2/
    └── ...
```

### 9.3 Writing Good Assertions

| Quality | Guidance |
|---|---|
| **Objectively verifiable** | "The output contains a column named `profit_margin`" — not "looks good" |
| **Descriptive names** | Text should be human-readable in the benchmark viewer |
| **Programmatic where possible** | Write a script to check, not eyeball — faster, reusable |
| **Avoid over-assertion** | Don't assert things that pass 100% in both with/without configurations — they add no signal |
| **Skip for subjective output** | Writing style, art quality — use human review instead |

```json
"expectations": [
  "The output is a .docx file in /mnt/user-data/outputs/",
  "The document contains a section titled 'Executive Summary'",
  "All monetary values are formatted as EUR with 2 decimal places",
  "The script generate.py was invoked (not custom ad-hoc code)"
]
```

---

## 10. JSON Schemas — All Types

### 10.1 `evals/evals.json`

```json
{
  "skill_name": "my-skill",
  "evals": [
    {
      "id": 1,
      "prompt": "User task prompt — realistic, specific, with context",
      "expected_output": "Human-readable description of success",
      "files": ["evals/files/sample_invoice.pdf"],
      "expectations": [
        "The output is a JSON file",
        "The file contains key 'total_amount'",
        "Currency is extracted as ISO 4217 code"
      ]
    }
  ]
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `skill_name` | string | ✅ | Must match frontmatter `name` |
| `evals[].id` | integer | ✅ | Unique per skill |
| `evals[].prompt` | string | ✅ | Realistic user message |
| `evals[].expected_output` | string | ✅ | Human-readable success definition |
| `evals[].files` | string[] | ❌ | Paths relative to skill root |
| `evals[].expectations` | string[] | ❌ | Verifiable assertions for grader |

### 10.2 `eval_metadata.json` (per eval run directory)

```json
{
  "eval_id": 0,
  "eval_name": "invoice-extraction-multipage",
  "prompt": "Extract all line items from the attached invoice PDF",
  "assertions": [
    "Output contains key 'line_items' as an array",
    "Each item has 'description', 'quantity', 'unit_price', 'total'"
  ]
}
```

### 10.3 `grading.json` (per run directory)

```json
{
  "expectations": [
    {
      "text": "The output is a .docx file",
      "passed": true,
      "evidence": "Found report.docx in outputs/ at step 4"
    },
    {
      "text": "Document contains 'Executive Summary' section",
      "passed": false,
      "evidence": "No heading matching 'Executive Summary' found in document XML"
    }
  ],
  "summary": {
    "passed": 5,
    "failed": 1,
    "total": 6,
    "pass_rate": 0.83
  },
  "execution_metrics": {
    "tool_calls": { "Read": 5, "Write": 2, "Bash": 8 },
    "total_tool_calls": 15,
    "total_steps": 6,
    "errors_encountered": 0,
    "output_chars": 12450,
    "transcript_chars": 3200
  },
  "timing": {
    "executor_duration_seconds": 165.0,
    "grader_duration_seconds": 26.0,
    "total_duration_seconds": 191.0
  }
}
```

**Critical**: Viewer expects `text`, `passed`, `evidence` — not `name`/`met`/`details`.

### 10.4 `metrics.json` (produced by executor inside `outputs/`)

```json
{
  "tool_calls": { "Read": 5, "Write": 2, "Bash": 8, "Edit": 1 },
  "total_tool_calls": 18,
  "total_steps": 6,
  "files_created": ["report.docx", "summary.json"],
  "errors_encountered": 0,
  "output_chars": 12450,
  "transcript_chars": 3200
}
```

### 10.5 `timing.json` (per run directory)

```json
{
  "total_tokens": 84852,
  "duration_ms": 23332,
  "total_duration_seconds": 23.3,
  "executor_start": "2026-05-27T10:30:00Z",
  "executor_end": "2026-05-27T10:32:45Z",
  "executor_duration_seconds": 165.0,
  "grader_start": "2026-05-27T10:32:46Z",
  "grader_end": "2026-05-27T10:33:12Z",
  "grader_duration_seconds": 26.0
}
```

**Important**: `total_tokens` and `duration_ms` come from the subagent task notification. They are **not persisted elsewhere** — capture them immediately when the notification arrives.

### 10.6 `benchmark.json`

```json
{
  "metadata": {
    "skill_name": "my-skill",
    "skill_path": "/path/to/my-skill",
    "executor_model": "claude-sonnet-4-20250514",
    "timestamp": "2026-05-27T10:30:00Z",
    "evals_run": [1, 2, 3],
    "runs_per_configuration": 3
  },
  "runs": [
    {
      "eval_id": 1,
      "eval_name": "invoice-extraction",
      "configuration": "with_skill",
      "run_number": 1,
      "result": {
        "pass_rate": 0.85,
        "passed": 6,
        "failed": 1,
        "total": 7,
        "time_seconds": 42.5,
        "tokens": 3800,
        "tool_calls": 18,
        "errors": 0
      }
    }
  ],
  "run_summary": {
    "with_skill": {
      "pass_rate":    { "mean": 0.85, "stddev": 0.05, "min": 0.80, "max": 0.90 },
      "time_seconds": { "mean": 45.0, "stddev": 12.0, "min": 32.0, "max": 58.0 },
      "tokens":       { "mean": 3800, "stddev": 400,  "min": 3200, "max": 4100 }
    },
    "without_skill": {
      "pass_rate":    { "mean": 0.35, "stddev": 0.08, "min": 0.28, "max": 0.45 },
      "time_seconds": { "mean": 32.0, "stddev": 8.0,  "min": 24.0, "max": 42.0 },
      "tokens":       { "mean": 2100, "stddev": 300,  "min": 1800, "max": 2500 }
    },
    "delta": {
      "pass_rate": "+0.50",
      "time_seconds": "+13.0",
      "tokens": "+1700"
    }
  },
  "notes": ["Eval 3 shows high variance — may be flaky"]
}
```

**Critical viewer field names**: `configuration` (not `config`), `result.pass_rate` (nested, not top-level).

### 10.7 `history.json` (workspace root, improve mode)

```json
{
  "started_at": "2026-05-27T10:30:00Z",
  "skill_name": "my-skill",
  "current_best": "v2",
  "iterations": [
    { "version": "v0", "parent": null,  "expectation_pass_rate": 0.65, "grading_result": "baseline", "is_current_best": false },
    { "version": "v1", "parent": "v0",  "expectation_pass_rate": 0.75, "grading_result": "won",      "is_current_best": false },
    { "version": "v2", "parent": "v1",  "expectation_pass_rate": 0.85, "grading_result": "won",      "is_current_best": true  }
  ]
}
```

### 10.8 Trigger Eval Set (for description optimisation)

```json
[
  {
    "query": "ok so my boss just sent me this xlsx file (Q4 sales final FINAL v2.xlsx) and she wants me to add a column that shows profit margin as a percentage",
    "should_trigger": true
  },
  {
    "query": "can you write a short poem about spreadsheets",
    "should_trigger": false
  }
]
```

20 queries recommended: 8–10 should-trigger, 8–10 should-not-trigger.  
Near-misses (adjacent domains, ambiguous phrasing) are the most valuable negatives.

### 10.9 `feedback.json` (from eval viewer)

```json
{
  "reviews": [
    { "run_id": "eval-0-with_skill", "feedback": "chart is missing axis labels", "timestamp": "..." },
    { "run_id": "eval-1-with_skill", "feedback": "",                             "timestamp": "..." },
    { "run_id": "eval-2-with_skill", "feedback": "perfect, love this",           "timestamp": "..." }
  ],
  "status": "complete"
}
```

Empty `feedback` = user approved that eval. Focus improvements on non-empty entries.

---

## 11. CLI Scripts Reference

All scripts are run from the **skill-creator directory** (the skill-creator skill ships these):

| Script | Command | Description |
|---|---|---|
| `package_skill` | `python -m scripts.package_skill <skill-folder>` | Produces distributable `.skill` file |
| `aggregate_benchmark` | `python -m scripts.aggregate_benchmark <workspace>/iteration-N --skill-name <name>` | Produces `benchmark.json` + `benchmark.md` |
| `generate_review` | `python eval-viewer/generate_review.py <workspace>/iteration-N` | Launches browser-based eval viewer |
| `generate_review` (static) | `python eval-viewer/generate_review.py <workspace>/iteration-N --static <output.html>` | Cowork mode: write static HTML instead of launching server |
| `generate_review` (with previous) | `python eval-viewer/generate_review.py <workspace>/iteration-N --previous-workspace <workspace>/iteration-N-1` | Shows side-by-side diff with previous iteration |
| `run_loop` | `python -m scripts.run_loop --eval-set <path> --skill-path <path> --model <id> --max-iterations 5 --verbose` | Full automated description optimisation loop |
| `run_eval` | `python -m scripts.run_eval --eval-set <path> --skill-path <path> --model <id>` | Single eval run (used internally by run_loop) |

### Kill the viewer server

```bash
kill $VIEWER_PID 2>/dev/null
```

---

## 12. Description Optimisation (run_loop)

Optimises the `description` field in SKILL.md frontmatter to maximise triggering accuracy.

### How it works

```
Split eval set: 60% train / 40% held-out test
         ↓
Evaluate current description (3 runs per query for reliability)
         ↓
Claude proposes improved description based on failures
         ↓
Re-evaluate on train + test
         ↓
Repeat up to --max-iterations
         ↓
Return best_description (selected by test score, not train — prevents overfitting)
```

### Output

```json
{
  "best_description": "Create, edit, read, fix, and transform Excel spreadsheets...",
  "best_score": 0.92,
  "iterations": [
    { "description": "...", "train_score": 0.70, "test_score": 0.65 },
    { "description": "...", "train_score": 0.85, "test_score": 0.88 },
    { "description": "...", "train_score": 0.90, "test_score": 0.92 }
  ]
}
```

**Apply result**: Replace `description` in SKILL.md frontmatter with `best_description`. Show before/after to the user with scores.

**Prerequisites**: Requires `claude -p` CLI tool — only available in Claude Code and Cowork, not Claude.ai.

---

## 13. Platform Differences Table

| Feature | Claude.ai | Claude Code | Cowork |
|---|---|---|---|
| Subagents (parallel runs) | ❌ | ✅ | ✅ |
| Browser / display | ❌ | ✅ | ❌ |
| `claude -p` CLI | ❌ | ✅ | ✅ |
| Description optimisation | ❌ Skip | ✅ Full | ✅ Full |
| Baseline comparison | ❌ Skip | ✅ Full | ✅ Full |
| Quantitative benchmarking | ❌ Skip | ✅ Full | ✅ Full |
| Eval viewer | Inline in chat | Browser | Static HTML → `--static` flag |
| Feedback collection | Inline Q&A | `feedback.json` from viewer | `feedback.json` download |
| Packaging (`package_skill.py`) | ✅ | ✅ | ✅ |
| Test case execution | Serial (one at a time) | Parallel subagents | Parallel subagents |
| Viewer via `generate_review.py` | Skip if no display | Browser launch | Use `--static <path>` |

### Claude.ai Adaptations

1. Run test cases **serially** — read SKILL.md, follow it, complete the task yourself
2. Skip baseline runs (no independent subagent)  
3. Skip quantitative benchmarking; rely on qualitative feedback
4. Skip description optimisation (no `claude -p`)
5. Present results directly in conversation, ask for inline feedback
6. File outputs: save to filesystem, tell user the path to download

### Cowork Adaptations

- Use `--static <output_path>` for the eval viewer (no browser)
- "Submit All Reviews" downloads `feedback.json` to `~/Downloads/` — check for multiple versions
- Description optimisation works (`run_loop.py` uses subprocess, not browser)
- Run tests in parallel; fall back to serial on severe timeout issues

---

## 14. Design Patterns

### Pattern 1 — Single-Script Skill

Use when the skill's core value is a deterministic transformation: convert, merge, extract, fill.

```
docx-formatter/
├── SKILL.md          ← Instructions: "Run scripts/format.py with these args"
└── scripts/
    └── format.py     ← Encodes all python-docx quirks
```

SKILL.md body:
```markdown
## Process
1. Read the user's document from the upload path.
2. Run the formatter:
   python scripts/format.py --input {path} --output /mnt/user-data/outputs/formatted.docx
3. Present the output file.
```

### Pattern 2 — Reference Router Skill

Use when a skill supports multiple domains or variants. Claude reads only the relevant variant's reference file — saving context for everyone else.

```markdown
## References
Based on the user's target platform:
- AWS → read `references/aws.md`
- GCP → read `references/gcp.md`  
- Azure → read `references/azure.md`
Do not read multiple reference files unless comparing platforms.
```

### Pattern 3 — Template-Driven Output

Use when output must match a fixed structure (reports, presentations, invoices).

```
report-generator/
├── SKILL.md
└── assets/
    └── report_template.docx
```

SKILL.md:
```markdown
## Output
Copy `assets/report_template.docx` to the output path, then populate using
the script. Do not create a new document from scratch — always start from
the template to preserve styles, margins, and page numbering.
```

### Pattern 4 — Progressive Reference Loading

Use when a skill has a large knowledge base that would be expensive to load every time.

```markdown
## Reference Files
This skill ships with detailed reference docs. Load them only when needed:

| File | Load when |
|---|---|
| `references/error_codes.md` | User reports an error or asks about a failure |
| `references/advanced_config.md` | User asks about edge cases or non-default config |
| `references/changelog.md` | User asks about version differences or upgrade paths |

For typical use cases, the guidance in this SKILL.md is sufficient.
```

### Pattern 5 — Eval-First Development

Write test prompts *before* writing the full skill body. Use them to drive the specification.

```
1. Ask: "What does a real user message for this task look like?"
2. Write 2–3 prompts → evals.json
3. Draft SKILL.md from the prompts (not the other way around)
4. Run → check if outputs match expected_output
5. Iterate SKILL.md; re-run
```

### Pattern 6 — Bundled Script as DRY Anchor

When running test cases, if **all 3 subagents independently write the same helper script** (e.g., `create_chart.py`), this is a signal to bundle it in `scripts/`. Write it once, reference it in SKILL.md. Every future invocation benefits.

---

## 15. Best Practices Checklist

### Authoring

- [ ] `name` in frontmatter matches the directory name exactly
- [ ] `description` covers all trigger scenarios, keywords, and edge cases
- [ ] Body is ≤ 500 lines; longer content moved to `references/`
- [ ] Large reference files (> 300 lines) include a table of contents
- [ ] Instructions use imperative voice
- [ ] Rationale is provided for non-obvious requirements
- [ ] Output format / template section is included if output structure matters
- [ ] Examples are included for complex or ambiguous inputs
- [ ] Reference files are pointed to with explicit "when to read" guidance
- [ ] The skill does not contain malware, exploit code, or misleading content

### Testing

- [ ] ≥ 2 realistic test prompts in `evals/evals.json`
- [ ] Prompts are specific and concrete (not "process a file")
- [ ] Assertions are objectively verifiable
- [ ] Both with-skill and baseline (without-skill or old-skill) runs executed
- [ ] Timing captured immediately from subagent notifications
- [ ] Human review completed via eval viewer before iterating
- [ ] Improvements generalise from feedback — not over-fitted to test cases
- [ ] Iteration continues until feedback is empty or progress stalls

### Description Optimisation (Claude Code / Cowork only)

- [ ] 20 trigger eval queries generated (10 should-trigger, 10 should-not-trigger)
- [ ] Near-miss negatives included (adjacent domains, ambiguous phrasing)
- [ ] User has reviewed and approved the eval set before running `run_loop`
- [ ] `best_description` (test score, not train score) applied to frontmatter
- [ ] Before/after scores shown to user

### Packaging & Distribution

- [ ] `python -m scripts.package_skill <skill-folder>` run from skill-creator dir
- [ ] `.skill` file presented to user via `present_files`
- [ ] Skill name preserved when updating an existing skill (no `-v2` suffix)
- [ ] Edited in a writable location (`/tmp/`) before packaging if original is read-only

---

## 16. Anti-Patterns to Avoid

| Anti-pattern | Why it's bad | Fix |
|---|---|---|
| Triggering logic in body (not description) | Claude won't read the body before deciding to trigger | Move all "when to use" to the `description` field |
| Overly rigid MUSTs and ALWAYSs | Brittle; model interprets intent poorly | Explain the *why*; let the model reason |
| SKILL.md > 500 lines | Context budget bloat on every invocation | Extract to `references/`; add pointer in SKILL.md |
| Test prompts that are too simple | Won't trigger the skill; tests nothing | Use realistic, multi-step, contextual prompts |
| Assertions that pass 100% with/without skill | No signal; waste of benchmark capacity | Remove or refine to be more discriminating |
| Reading all reference files unconditionally | Context waste; slows every invocation | Add "load when" conditions to reference pointers |
| Reinventing helper scripts on every run | Wastes tokens/time; inconsistent output | Bundle repetitive scripts in `scripts/` |
| Fitting skill to test cases, not general use | Useless outside those specific examples | Think: will this work for 1 million different inputs? |
| Appending `-v2` to skill name on update | Breaks install; user has to re-add | Preserve original `name`; the file content is the version |
| Using `configuration` field as `config` in benchmark.json | Viewer shows empty/zero values | Use exact field names from schema |

---

## 17. End-to-End Workflow Cheat Sheet

```
DECIDE ──────────────────────────────────────────────────────────────
│ What does the skill enable Claude to do?
│ When should it trigger?
│ What is the expected output format?
│ Does it need test cases? (Yes for verifiable output; maybe not for style)
│
RESEARCH ────────────────────────────────────────────────────────────
│ Interview user: edge cases, inputs, success criteria, dependencies
│ Check existing skills for overlap or reuse
│ Research APIs / libraries / docs if needed
│
DRAFT ───────────────────────────────────────────────────────────────
│ Write SKILL.md frontmatter (name + description)
│ Write body: workflow, output format, examples, reference pointers
│ Identify scripts to bundle; write them
│ Write 2–3 evals.json prompts
│
TEST ────────────────────────────────────────────────────────────────
│ Spawn with-skill AND baseline runs simultaneously
│ Draft assertions while runs execute
│ Grade runs → grading.json
│ Aggregate → benchmark.json
│ Launch eval viewer → human review → feedback.json
│
IMPROVE ─────────────────────────────────────────────────────────────
│ Read feedback.json (empty = approved; non-empty = fix)
│ Generalise improvements (don't overfit)
│ Explain the why; remove unnecessary rigidity
│ Bundle repeated scripts
│ Re-run from TEST step
│
OPTIMISE DESCRIPTION (Code/Cowork only) ────────────────────────────
│ Generate 20 trigger eval queries
│ User reviews + approves
│ Run: python -m scripts.run_loop --max-iterations 5 --verbose
│ Apply best_description to SKILL.md frontmatter
│
PACKAGE ─────────────────────────────────────────────────────────────
│ python -m scripts.package_skill <skill-folder>
│ present_files with .skill path
└─
```

---

## 18. Full Example: Minimal Skill

A simple skill with no scripts or references — just a well-written instruction set.

**Directory**: `commit-formatter/`

```yaml
---
name: commit-formatter
description: >
  Formats git commit messages to follow Conventional Commits specification.
  Use when the user asks to format, write, fix, or improve a git commit
  message, or says something like "help me commit this change" or "what
  should my commit message be". Also trigger for requests like "write a
  commit for X", "conventional commit style", or "semantic versioning commits".
---
```

```markdown
# Commit Formatter

Format git commit messages to follow the Conventional Commits 1.0 specification.

## Format

```
<type>(<scope>): <short description>

[optional body]

[optional footer(s)]
```

**Types**: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`, `perf`, `ci`, `build`, `revert`

- `feat`: new feature (correlates with MINOR in semver)
- `fix`: bug fix (correlates with PATCH in semver)
- `feat!` or `fix!` or any `!` suffix: breaking change (correlates with MAJOR)

## Rules
- Short description: present tense, lowercase, no period, ≤ 72 characters
- Scope: optional, lowercase, noun describing section of codebase
- Body: explain *what* and *why*, not *how*; wrap at 72 characters
- Footer: reference issues (`Closes #123`), breaking changes (`BREAKING CHANGE: ...`)

## Examples

**Input**: "Added user authentication with JWT tokens"
**Output**: `feat(auth): implement JWT-based user authentication`

**Input**: "Fixed the null pointer crash when user has no profile"
**Output**: `fix(profile): handle null profile reference on login`

**Input**: "Removed the deprecated /v1/users endpoint"
**Output**:
```
feat(api)!: remove deprecated /v1/users endpoint

BREAKING CHANGE: /v1/users has been removed. Use /v2/users instead.
```
```

---

## 19. Full Example: Multi-Domain Skill with Scripts

A realistic skill with variant routing, a bundled script, and reference files.

**Directory**: `cloud-cost-report/`

```yaml
---
name: cloud-cost-report
description: >
  Generate cost analysis reports from cloud billing exports (AWS, GCP, Azure).
  Use when the user uploads a billing CSV or asks to analyse cloud costs,
  spot anomalies, compare months, break down by service or tag, or produce
  a cost optimisation report. Also trigger for requests like "how much did
  we spend on X", "AWS Cost Explorer export", "Azure billing", "GCP billing
  data", or "where is our cloud budget going".
---
```

```markdown
# Cloud Cost Report

Generate structured cost analysis reports from cloud billing exports.

## Supported Platforms

| Platform | File type | Key columns |
|---|---|---|
| AWS | CSV | `service`, `usagetype`, `lineitem_unblendedcost` |
| GCP | CSV | `service.description`, `cost`, `project.id` |
| Azure | CSV | `ServiceName`, `Cost`, `ResourceGroup` |

## Process

1. Identify the cloud platform from the file headers.
2. Read the relevant reference doc for column mappings:
   - AWS → `references/aws_billing.md`
   - GCP → `references/gcp_billing.md`
   - Azure → `references/azure_billing.md`
3. Run the analysis script:
   ```
   python scripts/analyse.py \
     --platform {aws|gcp|azure} \
     --input {upload_path} \
     --output /mnt/user-data/outputs/cost_report.xlsx
   ```
4. Present the output file with a 3-sentence summary:
   - Total spend for the period
   - Top 3 cost drivers
   - One actionable optimisation recommendation

## Output Format

The Excel report always contains these sheets:
- **Summary**: total cost, period, platform, record count
- **By Service**: cost per service, sorted descending
- **By Month**: month-over-month trend
- **Anomalies**: items > 2σ above mean for their service

## Reference Files

Platform reference files contain column name mappings and known anomalies
specific to each provider's billing export format. Load only the file that
matches the detected platform — do not load all three.
```

**`scripts/analyse.py`** (abbreviated):

```python
#!/usr/bin/env python3
"""Cloud billing cost analyser. Produces XLSX report."""
import argparse
import pandas as pd
from openpyxl import Workbook

PLATFORM_COLUMNS = {
    "aws":   {"service": "lineitem_productcode", "cost": "lineitem_unblendedcost"},
    "gcp":   {"service": "service.description",  "cost": "cost"},
    "azure": {"service": "ServiceName",           "cost": "Cost"},
}

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--platform", required=True, choices=["aws", "gcp", "azure"])
    parser.add_argument("--input",    required=True)
    parser.add_argument("--output",   required=True)
    args = parser.parse_args()

    cols = PLATFORM_COLUMNS[args.platform]
    df   = pd.read_csv(args.input)
    df   = df.rename(columns={cols["service"]: "service", cols["cost"]: "cost"})
    df["cost"] = pd.to_numeric(df["cost"], errors="coerce").fillna(0)

    with pd.ExcelWriter(args.output, engine="openpyxl") as writer:
        summary = pd.DataFrame({"Metric": ["Total", "Records"], "Value": [df["cost"].sum(), len(df)]})
        summary.to_excel(writer, sheet_name="Summary", index=False)
        df.groupby("service")["cost"].sum().sort_values(ascending=False).to_excel(writer, sheet_name="By Service")

if __name__ == "__main__":
    main()
```

**`evals/evals.json`**:

```json
{
  "skill_name": "cloud-cost-report",
  "evals": [
    {
      "id": 1,
      "prompt": "Here's our AWS CUR export from April. Can you give me a cost breakdown and flag anything that looks unusual?",
      "expected_output": "XLSX report with Summary, By Service, By Month, Anomalies sheets; 3-sentence summary in chat",
      "files": ["evals/files/aws_cur_april.csv"],
      "expectations": [
        "Output file is an .xlsx in /mnt/user-data/outputs/",
        "Workbook contains sheet named 'By Service'",
        "Workbook contains sheet named 'Anomalies'",
        "Chat response includes a 3-sentence summary",
        "scripts/analyse.py was invoked with --platform aws"
      ]
    }
  ]
}
```

---

*Reference card complete. For the description optimisation loop, `run_loop.py` scripts, and the eval viewer, see the bundled `skill-creator` skill. For platform-specific packaging details, run `python -m scripts.package_skill --help` from the skill-creator directory.*
