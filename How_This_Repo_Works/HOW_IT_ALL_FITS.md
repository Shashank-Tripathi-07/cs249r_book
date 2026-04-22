# How It All Fits Together

> **Purpose:** The master view. How the pieces connect, how CI/CD is organized, how
> deployment works, and what to check when something is broken. Read REPO_OVERVIEW.md,
> then the component docs, then this one.

---

## The Full Picture

```
                         ┌─────────────────────────────────────────────┐
                         │              mlsysbook.ai                   │
                         │  (Cloudflare Pages or Vercel -- the web)    │
                         └──────────────┬──────────────────────────────┘
                                        │  (links to all sub-sites)
          ┌─────────────────────────────┼────────────────────────────────┐
          │                             │                                 │
          v                             v                                 v
  mlsysbook.ai/book          mlsysbook.ai/tinytorch         mlsysbook.ai/staffml
  (Quarto static site)       (Quarto static site)           (Next.js app)
  Built from: book/          Built from: tinytorch/         Built from: interviews/staffml/
          │                             │                                 │
          │                             │                                 v
          │                             │                     staffml-vault-worker
          │                             │                     (Cloudflare Worker -- question API)
          │                             │                                 │
          │                             │                                 v
          │                             │                         Cloudflare D1
          │                             │                     (SQLite cloud database)
          │                             │                     Populated from: vault/vault.db
          │
  mlsysbook.ai/labs                         (labs are part of the book deploy path)
  (Marimo + WASM)
  Built from: labs/

          Everything uses shared CSS/JS synced from: shared/
```

---

## The Six Subsites

Each component of the curriculum has its own deployed subsite:

```
book/          → mlsysbook.ai            The textbook
tinytorch/     → mlsysbook.ai/tinytorch  TinyTorch course site
labs/          → mlsysbook.ai/labs       Interactive labs
interviews/    → mlsysbook.ai/staffml    Interview prep platform
slides/        → mlsysbook.ai/slides     Lecture slides
kits/          → mlsysbook.ai/kits       Hardware kit exercises
```

Plus `site/` which is the landing page at mlsysbook.ai root that links to all of them.

---

## How mlsysim Connects to Everything

mlsysim is the glue. It's the "single source of truth for hardware numbers" and it flows
into three different places via three different mechanisms:

```
mlsysim/
   │
   ├── Imported by labs at runtime
   │     When: a student runs a lab in the browser
   │     How: compiled to a .whl file, installed via micropip in WASM
   │     Usage: mlsysim.Hardware.Cloud.H100.compute.peak_flops.m_as("TFLOPs/s")
   │
   ├── Referenced by StaffML (indirectly)
   │     When: napkin-math questions in the vault use real hardware specs
   │     How: the humans who wrote the YAML used mlsysim to generate the numbers
   │     Note: not a live import -- numbers are baked into the YAML at authoring time
   │
   └── Used in the roofline and simulator tools in staffml/src/app/
         When: a user opens the interactive roofline calculator in StaffML
         How: direct Python import if running locally, WASM if running in browser
```

---

## How Shared Assets Flow

```
shared/                 ← SOURCE OF TRUTH, edit here
   ├── scripts/subscribe-modal.js
   ├── assets/img/logo-seas-shield.png
   └── styles/_brand.scss, themes/...
          │
          │  bash shared/scripts/sync-mirrors.sh
          ▼
book/quarto/assets/scripts/subscribe-modal.js    (copy)
book/quarto/assets/images/icons/logo-seas-shield.png
book/quarto/assets/styles/_brand.scss
interviews/staffml/public/logo-seas-shield.png   (copy)
labs/assets/scripts/subscribe-modal.js           (copy)
site/assets/scripts/subscribe-modal.js           (copy)
kits/assets/scripts/subscribe-modal.js           (copy)
mlsysim/docs/scripts/subscribe-modal.js          (copy)
```

**Critical rule:** Never edit the mirror copies directly -- your changes will be
silently overwritten. Always edit in `shared/` then run the sync script.

If CI fails with "mirror stale", someone edited a mirror instead of the source.

---

## CI/CD: The Workflow System (53 files)

The `.github/workflows/` directory has 53 workflow files. They follow a naming convention:

```
{component}-{action}-{environment}.yml

Components: book, tinytorch, labs, staffml, mlsysim, slides, kits, site, instructors, infra
Actions:    validate, preview, publish, build (pdfs), update (pdfs), welcome, auto-pr
Environments: dev, live
```

Each component has up to four workflows:

```
{component}-validate-dev.yml    Runs on PR. TypeCheck, tests, build check. Fast.
                                No deploy.

{component}-preview-dev.yml     Runs on merge to dev. Deploys to a preview URL.
                                Used for team review before publishing.

{component}-publish-live.yml    Runs manually or on tag. Deploys to production.

{component}-build-pdfs.yml      Generates PDFs. Separate from deploy.
```

Plus infrastructure workflows that apply across the whole repo:

```
infra-health-check.yml         Pings live sites to check they're up
infra-link-check.yml           Checks internal links aren't broken
infra-link-rot-nightly.yml     Nightly check for broken external links
infra-publish-guard.yml        Gate that prevents publishing without validation
infra-container-linux.yml      Builds Docker images for Quarto environments
ci-sanity.yml                  Quick sanity check (runs on every PR)
```

---

## The PR + Deploy Flow

```
STEP 1: You branch off dev
  git checkout dev
  git pull
  git checkout -b feat/my-change

STEP 2: You make changes and open a PR targeting dev
  gh pr create --base dev

STEP 3: CI runs validate workflows (triggered by pull_request event)
  staffml-validate-dev.yml     if interviews/staffml/** changed
  tinytorch-validate-dev.yml   if tinytorch/** changed
  labs-validate-dev.yml        if labs/** changed
  ci-sanity.yml                always

STEP 4: PR is reviewed and merged to dev

STEP 5: Merge triggers preview workflows
  staffml-preview-dev.yml      deploys to staffml-preview.pages.dev
  book-preview-dev.yml         deploys to book-preview.pages.dev
  etc.

STEP 6: Maintainer reviews the preview

STEP 7: Maintainer manually triggers publish workflow
  staffml-publish-live.yml     deploys to mlsysbook.ai/staffml
```

The preview step is important. There's always a chance the build succeeds in CI but
looks broken in the actual deployed environment. Preview URLs let maintainers verify
before touching production.

---

## Path-Based Triggering

Workflows only run when relevant files change. This is controlled by the `paths:` filter:

```yaml
# From staffml-validate-dev.yml:
on:
  pull_request:
    paths:
      - 'interviews/staffml/**'
      - '.github/workflows/staffml-validate-dev.yml'
```

This means:
- Changing only `tinytorch/` doesn't run the StaffML tests
- Changing a workflow file itself triggers its own validation run
- Changing `shared/` triggers... nothing directly. The shared sync CI check lives
  in whichever component workflows have shared asset checks.

**Common gotcha:** If you change `mlsysim/` and labs import it, the labs tests might
not automatically run because labs paths didn't change. Always check if you need to
manually trigger downstream workflows.

---

## What Deploys Where

```
QUARTO STATIC SITES (book, tinytorch, slides, kits)
  Built by: quarto render
  Hosted on: Cloudflare Pages
  Artifacts: static HTML/CSS/JS files

MARIMO LABS
  Built by: marimo export (generates WASM bundles)
  Hosted on: Cloudflare Pages (same as book)
  Artifacts: HTML files with embedded WASM

STAFFML NEXT.JS APP
  Built by: next build (static export mode)
  Hosted on: Cloudflare Pages
  Artifacts: static HTML/JS/CSS + API calls hit the worker

STAFFML VAULT WORKER
  Built by: wrangler deploy
  Hosted on: Cloudflare Workers (edge)
  Artifact: single compiled JS file

CLOUDFLARE D1 DATABASE
  Populated by: vault ship (runs migration SQL)
  Hosted on: Cloudflare D1 (managed SQLite)
  Updated when: vault YAML files change and a new release is shipped
```

---

## The "dev Is Production" Model

There's no `main` branch. This is intentional.

```
dev branch = the stable, always-deployable branch
feature branches = short-lived, branch off dev, merge back to dev
```

No release branches. No staging environment between dev and production (the preview
deploy serves that purpose). No git tags for most changes (vault releases use tags;
the rest doesn't).

**Why?** The curriculum is continuously iterated, not versioned. The textbook, labs,
and StaffML don't have "v2.0 release events" -- they're always just the current state.

---

## Dependency Danger Zones

These are the places where a change in one component can silently break another:

```
1. mlsysim API changes
   Risk: labs call mlsysim functions directly. If a function is renamed or returns
   different units, labs fail at runtime in users' browsers.
   Test: run the labs locally after any mlsysim change.

2. vault schema changes
   Risk: vault/schema/ feeds TypeScript types (staffml app), SQL DDL (worker),
   and Python validation (vault-cli). Changing the schema requires regenerating all
   three and deploying them atomically.
   Test: run `vault codegen --check` after any schema change.

3. shared asset edits
   Risk: editing a mirror copy instead of shared/. Silently reverts on next sync.
   Test: run `bash shared/scripts/sync-mirrors.sh --check` before committing.

4. TinyTorch src/ vs modules/ drift
   Risk: if src/ is updated but modules/ template isn't regenerated, students
   starting new modules get stale templates.
   Test: the tinytorch-validate workflow checks this.

5. vault corpus.json edited directly
   Risk: corpus.json is a generated file. Direct edits get overwritten on next compile.
   The vault pre-commit hook guards this.
```

---

## Where to Look When Something Is Broken

```
BROKEN SITE (staffml, labs, book)
  1. Check latest GitHub Actions run for the publish workflow
  2. Check the preview deploy (was it broken before publish?)
  3. Check recent merges to dev that touched that component

LABS NOT LOADING IN BROWSER
  1. Check browser console for WASM errors
  2. Is the mlsysim .whl present in labs/assets/wheels/?
  3. Is the version in the whl filename matching what the lab imports?

STAFFML QUESTIONS NOT LOADING
  1. Is the vault worker up? (hit the worker URL directly)
  2. Is D1 accessible? (check Cloudflare dashboard)
  3. Is the worker's schema fingerprint matching D1? (check worker logs)

TINYTORCH TESTS FAILING
  1. Find the module -- tests are in tinytorch/tests/XX_name/
  2. Check if src/ changed since the last passing test
  3. Run the specific test file with pytest

CI FAILING ON "MIRROR STALE"
  1. Run: bash shared/scripts/sync-mirrors.sh
  2. Commit the updated mirror files
  3. The failure means someone edited shared/ without running the sync
```
