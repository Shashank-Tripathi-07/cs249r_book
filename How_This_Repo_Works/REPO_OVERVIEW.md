# MLSysBook Repository -- Overview Guide

> **Purpose of this document:** Explain what the whole repository is, how it's divided,
> what each part does, and how they connect. Start here before reading any other doc.

---

## What Is This Repo?

This is the source for an entire university curriculum called **ML Systems** (Harvard CS249r),
maintained by Prof. Vijay Janapa Reddi. The stated goal: teach 100,000 learners how to
*engineer* AI systems, not just use them.

The key design decision is that **everything lives in one repo**. That's intentional.
The textbook, the coding framework, the interactive labs, the mock interview platform --
they're all one integrated curriculum. A student is supposed to move through them in sequence.

---

## The Big Picture: What Lives Where

```
cs249r_book/
|
|-- book/              The textbook itself (Quarto/Markdown source)
|                      Compiled to mlsysbook.ai
|
|-- tinytorch/         Build-your-own ML framework (Python, NumPy only)
|                      20 progressive coding modules
|                      Companion to the textbook chapters
|
|-- labs/              33 interactive browser experiments (Marimo notebooks)
|                      Run via WebAssembly -- no install needed
|                      Bridge between reading and coding
|
|-- interviews/        StaffML -- mock interview prep platform
|   |-- staffml/           Next.js web app (the UI)
|   |-- staffml-vault-worker/   Cloudflare Worker (question API)
|   |-- vault/             The question corpus (9,113 YAML files)
|   |-- vault-cli/         CLI tool to manage questions
|
|-- mlsysim/           MLSys simulation library (Python)
|                      Physics-grounded hardware models
|                      Powers the interactive calculations in labs
|
|-- shared/            Assets shared across subsites
|                      CSS, logos, scripts -- synced with a script
|
|-- slides/            Lecture slides (Quarto source)
|-- kits/              Hardware kit exercises (Arduino/embedded)
|-- instructors/       Instructor-only materials
|-- site/              The mlsysbook.ai landing site
|-- .github/workflows/ All CI/CD pipelines (40+ files)
```

---

## What Came First -- The Build Order

Understanding what depends on what prevents a lot of confusion.

```
LAYER 0 -- The Foundation
  book/          The textbook. Everything else is an extension of this.
  mlsysim/       Hardware simulation library. Labs and StaffML both use it.

LAYER 1 -- Learning Tools (depend on book existing)
  tinytorch/     Coding exercises tied to textbook chapters.
  labs/          Interactive labs tied to textbook chapters.
                 Labs IMPORT mlsysim to run simulations.

LAYER 2 -- Career Tools (depends on the subject matter existing)
  interviews/    StaffML interview prep.
                 Questions are grounded in the same ML systems theory.
                 Uses mlsysim hardware specs for napkin-math questions.

LAYER 3 -- Shared Infrastructure
  shared/        Logos, CSS, JS snippets used by book, labs, site.
  site/          The mlsysbook.ai homepage that links everything.
```

If you change `mlsysim`, labs might break (they import it). If you change the vault
schema in `interviews/vault/`, the StaffML worker and UI might break.

---

## The Three Audiences

Different people interact with different parts of the repo:

```
STUDENTS (coding TinyTorch)
  Entry: tinytorch/modules/  -- their workspace
  They use: tinytorch/src/ (reference), tito CLI (workflow)
  They never touch: labs/, interviews/, book/

STUDENTS (doing labs)
  Entry: labs/vol1/ and labs/vol2/ -- open in browser
  They use: nothing else directly
  Behind the scenes: mlsysim is bundled in via WASM

JOB SEEKERS (using StaffML)
  Entry: mlsysbook.ai/staffml
  They use: the web app only
  Never see: vault YAML files, workers, any code

CONTRIBUTORS / MAINTAINERS
  Touch everything
  Add questions: interviews/vault/questions/
  Fix bugs: anywhere
  Add content: book/, tinytorch/src/, labs/vol1/ or vol2/
```

---

## How the Pieces Talk to Each Other at Runtime

This is the part that trips people up. Here's what's actually happening when
different parts of the system run:

```
TINYTORCH (runs locally on student's machine)
  Student runs: tito module start 01_tensor
  tito reads:   tinytorch/src/01_tensor/  (the reference implementation)
  Student edits: tinytorch/modules/01_tensor/  (their copy)
  Tests run against: student's code via pytest
  No network calls. Fully offline.

LABS (runs in browser)
  Student opens: mlsysbook.ai/labs/vol1/lab_05_nn_compute
  Browser downloads: a Marimo WASM bundle
  Bundle contains: mlsysim (compiled to WebAssembly via Pyodide)
  Lab cells call: mlsysim.Hardware.H100.compute.peak_flops.m_as("TFLOPs/s")
  That returns: real hardware specs from mlsysim's registry
  No server needed after initial load. Runs 100% client-side.

STAFFML (runs as a web app with backend)
  User opens: mlsysbook.ai/staffml
  Browser talks to: Next.js app (hosted on Vercel/Cloudflare Pages)
  Next.js talks to: staffml-vault-worker (Cloudflare Worker)
  Worker queries: Cloudflare D1 database (SQLite in the cloud)
  D1 was populated from: vault/questions/*.yaml (compiled offline)
  For AI features: worker calls Gemini API
  User data (progress, scores): stored in browser localStorage only
```

---

## The Shared Assets Problem

Several subsites (book, labs, staffml, site) need the same files -- logos, CSS,
subscribe modal JavaScript. The repo uses real file copies (not symlinks) because
Quarto's build system doesn't follow symlinks correctly.

The `shared/` folder is the source of truth. After editing anything there, you run:

```bash
bash shared/scripts/sync-mirrors.sh
```

This copies files to all the places that need them. If CI fails with "mirror stale",
it means someone edited a shared asset but forgot to run the sync script.

---

## The `dev` Branch Is Production

There's no `main` branch. All work happens on `dev`. The workflow is:

1. Branch off `dev` for your change
2. Open a PR targeting `dev`
3. CI runs validate workflows (one per component)
4. Merge to `dev`
5. Separate "publish" workflows deploy to live sites

Never push directly to `dev`. PRs only.
