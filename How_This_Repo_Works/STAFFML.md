# StaffML -- Deep Dive

> **Purpose:** Explain how StaffML is built, how questions flow from YAML files to the user's
> browser, what each service does, and how the four deployment tracks and six mastery levels
> work. Read REPO_OVERVIEW.md first.

---

## What Is StaffML?

StaffML is a free interview prep platform for ML systems engineers. It hosts 9,113
physics-grounded questions across four deployment tracks (Cloud, Edge, Mobile, TinyML) at
six mastery levels. Students can browse questions, do timed mock interviews (Gauntlet), and
track their coverage across competency areas.

The key design principle: questions are grounded in real hardware physics. A "napkin math"
question about H100 memory bandwidth uses the actual numbers from mlsysim, not made-up round
numbers. This makes the questions useful for real interviews, not just trivia.

---

## Architecture Overview

```
vault/questions/*.yaml    9,113 YAML source files -- the corpus
      |
      v  (vault-cli: compile & publish)
      |
vault/vault.db            SQLite database (built artifact)
      |
      v  (vault ship: deploy to Cloudflare)
      |
Cloudflare D1             SQLite-in-the-cloud (production read replica)
      |
      v
staffml-vault-worker/     Cloudflare Worker (the API)
      |  GET /questions?track=cloud&level=3
      |  GET /questions/:id
      |  GET /chains/:id
      |  POST /search
      v
staffml/ (Next.js app)    The UI
      |
      v
User's browser            Practice, Gauntlet, Progress, Plans
```

There are also TWO other workers in the system:
- **staffml-vault-worker:** The primary question API (described above)
- The main staffml app also talks to Gemini (Google's AI) for AI-assisted features

---

## The Question Corpus Pipeline

This is the most important thing to understand about StaffML's architecture.

### 1. Source: YAML files

Each question is its own YAML file in `vault/questions/`:

```
vault/questions/
  cloud/
    q_cloud_memory_001.yaml
    q_cloud_memory_002.yaml
    ...
  edge/
    q_edge_thermal_001.yaml
    ...
  mobile/
  global/      (questions relevant across all tracks)
```

A question file looks like this:
```yaml
id: q_cloud_memory_001
track: cloud
level: 3              # Bloom's taxonomy level (1-6+)
topic: memory_bandwidth
title: "H100 effective memory bandwidth under fragmentation"
stem: |
  An H100 GPU has a theoretical peak memory bandwidth of 3.35 TB/s.
  In a production transformer serving workload, you observe sustained
  2.1 TB/s. What is the bandwidth utilization, and what are two common
  causes of the gap?
answer: |
  Utilization: 2.1 / 3.35 = 62.7%. Common causes: (1) irregular access
  patterns from variable-length sequences causing non-coalesced reads,
  (2) KV-cache fragmentation from different sequence lengths in a batch.
tags: [memory, bandwidth, serving, kv-cache]
authors: [profvjreddi]
status: published
```

Why per-file YAML instead of one big JSON?
- Reviewable in PRs (one question = one diff)
- Debuggable with just a text editor
- Git history tracks who added/changed each question
- Merge conflicts are per-question, not full-corpus conflicts

### 2. Build: vault-cli

The `vault-cli/` tool compiles YAML files into a SQLite database:

```bash
vault generate        # Regenerate all questions from YAML
vault validate        # Check schema compliance
vault compile         # Build vault.db from all YAML files
vault codegen         # Generate TypeScript types from LinkML schema
vault ship            # Atomic deploy: D1 + worker + paper macros
```

The compiled `vault.db` has:
- `questions` table with all fields
- `chains` and `chain_questions` tables for question sequences
- `tags` table
- FTS5 full-text search virtual table (with auto-sync triggers)

`corpus.json` is also generated from the YAML -- it's a 28MB JSON blob that is the
**generated** representation. Do not edit it directly; it will be overwritten on the
next compile.

### 3. Serve: Cloudflare D1 + Worker

The compiled vault.db is deployed to Cloudflare D1 -- Cloudflare's managed SQLite.

The `staffml-vault-worker` (a Cloudflare Worker) sits in front of D1 and exposes the API:
- Handles filtering by track, level, topic
- Rate limiting (per `rate_limit.ts`)
- AI features route to Gemini
- Returns JSON

The worker is globally distributed (Cloudflare edge). Queries run at the edge node
closest to the user.

### 4. UI: Next.js App

The `staffml/` directory is a Next.js app deployed to Cloudflare Pages (or Vercel).
It talks to the vault worker for all question data.

User progress (what questions you've answered, your scores, gauntlet history) is stored
in localStorage -- not on any server. This means:
- No user accounts, no login required
- Progress can be exported/imported via JSON
- Clearing browser data loses all progress

---

## The Four Deployment Tracks

Each track represents a different operating context with fundamentally different constraints:

```
CLOUD        Data center training & serving
  Primary constraint: Memory bandwidth and network interconnect
  Key questions: KV-cache sizing, pipeline parallelism, interconnect topology,
                 serving throughput vs latency tradeoffs

EDGE         Autonomous vehicles, robotics, drones
  Primary constraint: Thermal envelope and real-time latency
  Key questions: Thermal-aware scheduling, deadline-constrained inference,
                 redundancy for safety-critical systems

MOBILE       On-device AI for smartphones/tablets
  Primary constraint: Battery life and shared hardware resources
  Key questions: Neural engine vs CPU vs GPU routing, power budget,
                 sharing memory with camera/audio subsystems

TINYML       Microcontrollers, sensors, ultra-low-power
  Primary constraint: SRAM capacity (often <512KB) and hard real-time
  Key questions: Model fitting in SRAM, fixed-point arithmetic,
                 wake-word detection with mW power budgets
```

---

## The Six Mastery Levels

Questions are tagged L1-L6+ following Bloom's Taxonomy:

```
L1  RECALL      Define, name, list. "What is FLOP?"
L2  UNDERSTAND  Explain, describe. "Explain why attention is O(n^2)"
L3  APPLY       Use a concept. "Calculate inference latency for ResNet50 on H100"
L4  ANALYZE     Break down, compare. "Compare pipeline vs tensor parallelism tradeoffs"
L5  EVALUATE    Critique, justify. "Given these constraints, which approach is better?"
L6+ CREATE      Design, architect. "Design a serving system for a 70B parameter model
                                    handling 10,000 requests/second at P99 < 100ms"
```

L6+ is the "Staff engineer" level -- these questions have no single correct answer.
They're design problems that test whether you can reason through a complex tradeoff space.
StaffML's Chains feature sequences questions from L1 to L6+ on a single topic so you
can drill from recall through architecture.

---

## The UI Pages (What the User Sees)

```
staffml/src/app/

  page.tsx              Landing page / dashboard
  welcome/              Onboarding flow (shown on first visit)
  framework/            The competency framework explanation
  practice/             Practice mode: answer one question at a time,
                        spaced repetition based on your score history
  gauntlet/             Timed mock interview: N questions, timer, self-assessment
                        keyboard shortcuts (1-5 scoring, Space for next)
  progress/             Coverage heatmap across tracks and levels
  plans/                Study plans (curated sequences by job target)
  roofline/             Interactive roofline calculator (uses mlsysim)
  simulator/            Hardware simulation tool
  dashboard/            Your stats overview
  contribute/           How to add questions
  about/                About page
```

The Gauntlet is the main practice mode. It simulates a real interview: questions appear
one at a time, you answer verbally/mentally, then rate yourself 1-5 on how well you did,
then move to the next. Session length is configurable (15/30/45 min).

---

## How to Add Questions to the Vault

1. Find the right directory: `vault/questions/cloud/`, `edge/`, `mobile/`, or `global/`
2. Create a new YAML file following the schema in `vault/schema/staffml_taxonomy.yaml`
3. Run `vault validate` to check your file passes schema
4. Run `vault compile` to rebuild vault.db locally
5. Test with `vault api` (local worker shim -- no Cloudflare needed)
6. Open a PR targeting the `dev` branch

The vault-cli does not require a Cloudflare account for local development. The `vault api`
command starts a local HTTP server that serves questions from your local vault.db.

---

## How to Run StaffML Locally

```bash
# Prerequisites: Node.js 20+, wrangler (Cloudflare CLI)

# 1. Build the vault
cd interviews/vault-cli
pip install -e .
vault compile    # Creates vault.db

# 2. Start the worker locally
cd interviews/staffml-vault-worker
npm install
wrangler dev     # Serves worker at localhost:8787

# 3. Start the Next.js app
cd interviews/staffml
npm install
NEXT_PUBLIC_VAULT_API_URL=http://localhost:8787 npm run dev
# App at localhost:3000
```

---

## The Vault Schema System

Question schema is defined once in `vault/schema/staffml_taxonomy.yaml` (LinkML format).
From this single source, the vault-cli generates:
- Pydantic models (Python validation)
- SQL DDL (the database table structure)
- TypeScript types (used by the Next.js app)

This means you can't get out of sync: if you add a new field to the LinkML schema, running
`vault codegen` updates all three consumers. The CI check `vault codegen --check` fails if
any generated file is stale.

---

## What Can Break and Why

**1. Corpus / question count discrepancy**
The site filters by `status = "published"` and the paper filters by `validated = true`.
These are different sets. Currently ~1,100 questions are in one but not the other.
Don't report a question count mismatch as a bug unless you've checked both filters.

**2. Design Ledger import/export**
The `importProgress()` function in `staffml/src/lib/progress.ts` does an atomic
snapshot-validate-write-rollback. If any question in an import file has the wrong shape,
the entire import is rejected and localStorage is left untouched. This is intentional.

**3. Gauntlet keyboard handler stale closure**
Fixed in PR #1430: `scoreAndNext` must be wrapped in `useCallback` and added to the
keyboard `useEffect` dependency array, or the handler captures stale score state.

**4. D1 schema mismatch**
The worker checks `sqlite_master` on cold start and hashes the actual DDL against the
expected fingerprint. On mismatch, it drops into degraded read-only mode (serves from
Cache API cache) rather than serving 500 errors.
