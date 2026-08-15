# Testing TinyTorch: Lessons from SQLite's Testing Philosophy

This document adapts SQLite's public testing methodology ([sqlite.org/testing.html](https://sqlite.org/testing.html)) to TinyTorch. SQLite is one of the most heavily tested pieces of software in existence (100% branch coverage, multiple independent test harnesses, millions of test cases), and while TinyTorch is an educational Python ML framework rather than an embedded C database engine, most of the underlying testing principles translate directly.

This document has two purposes:

1. Record what's already been done, so it isn't re-derived from scratch next time.
2. Lay out what's left, mapped from SQLite's technique list to a TinyTorch-specific equivalent, so future work has a menu instead of a blank page.

## Already done

### MC/DC (Modified Condition/Decision Coverage)

A full MC/DC audit was run across all 20 `src/` modules plus the milestone grading/gating logic. For every compound boolean decision (`and`/`or`/`not` combinations, chained comparisons), the audit checked whether the test suite:

- exercises every entry/exit point (every `raise`, every early return)
- exercises the decision's outcome both True and False
- exercises each individual condition both True and False
- has a pair of test cases where only one condition changes and the decision's outcome changes correspondingly (true condition independence, not just decision coverage)

Every finding was then independently reproduced against a clean `dev` checkout (not just re-read from source) to confirm it was real and to confirm whether the underlying code was actually correct or actually broken. That distinction mattered: most findings were coverage gaps on correct code, some were real bugs.

Result: ~37 confirmed coverage gaps got regression tests (5 PRs, organized by module group: core tensor/autograd, dataloader/training, architecture modules, systems/perf, milestones), and 8 confirmed real bugs got fixed with dedicated regression tests (8 separate PRs, one per bug). See PR history on `dev` for specifics; each PR body documents the exact defect and fix.

### Regression testing for every fixed bug

Every one of the 8 bug fixes above shipped with a test reproducing the original failure mode. This is SQLite's practice of never fixing a bug without adding a permanent test for it, carried over directly.

## Not yet done, mapped from SQLite's technique list

| SQLite technique | TinyTorch equivalent | Why it would catch something new |
|---|---|---|
| **Malformed input file tests** (a corrupted database file must never crash SQLite further) | Corrupt `.tito/progress.json`, `.tito/milestones.json`, checkpoint files, tokenizer vocab files, and confirm graceful failure rather than a crash or silent data loss | Zero coverage today. The module start/resume deadlock bug and the untested `can_unlock` gate both live in exactly this territory: state files that can desync or be malformed |
| **Fuzz testing** (AFL, dbsqlfuzz) | Feed `Tensor()`, `.reshape()`, tokenizer `.encode()`, and milestone JSON parsing random or malformed input, via `hypothesis` or plain randomized fuzzing | Cheap to set up, finds the input nobody thought to write a test for. Complements MC/DC rather than replacing it, SQLite explicitly notes fuzzing and 100% MC/DC find different classes of bugs |
| **Boundary value tests** | Systematic sweep: `batch_size=0/1`, empty tensors, `vocab_size` boundary, single-token sequences, 1-layer networks, kernel size equal to input size | Only hit a few of these by accident this round (`Tensor([])`). Never done as a deliberate, systematic pass |
| **Mutation testing** | `mutmut` or `cosmic-ray` run against the 20 `src/` modules | Would have surfaced most of this round's MC/DC gaps automatically, without a manual condition-by-condition audit. Good complement, not a replacement, mutation testing tells you the suite is weak, MC/DC tells you exactly where and why |
| **Coverage measurement (gcov)** | Turn on `pytest-cov`, already an installed dependency and currently unused anywhere in CI | A one-line CI fix identified in an earlier audit pass, not yet acted on |
| **Cross-checking against an independent reference** (SQLite's SLT tests it against other SQL engines for identical results) | Cross-check TinyTorch's `Tensor` ops against real PyTorch/NumPy on identical random inputs: `assert np.allclose(tinytorch_op(x), torch_op(x))` | Highest-value item on this list. TinyTorch reimplements operations that have an actual ground truth sitting right there. This is the one class of numerical bug that unit tests with hand-picked expected values structurally cannot catch |
| **Resource/leak testing** | Repeated `tito module complete` cycles, DataLoader multiprocessing workers, subprocess handles held by `tito system jupyter` | Already observed directly: orphaned multiprocessing workers survived a timeout during manual testing earlier in this project. Never systematically tested |
| **Anomaly/fault-injection testing** (OOM, I/O error, simulated crash mid-operation) | Kill `tito` mid-export, interrupt a milestone run mid-write, simulate a full disk during checkpoint save | Same bug category as the module start/resume deadlock already found and fixed. Likely more bugs of this shape remain |
| **Testing under varied configurations** | Python 3.10-3.13 (CI currently only runs 3.11), with and without optional dependencies | Already flagged in an earlier CI audit, not yet acted on |

## Doesn't translate

Valgrind/UBSan-style undefined-behavior detection, disabled-optimization builds, and most of SQLite's C-specific dynamic analysis don't have a meaningful TinyTorch equivalent, Python doesn't have those failure modes. The nearest analog is numerical: NaN/Inf propagation, which an earlier audit found untested in 16 of the 20 modules. That gap is effectively TinyTorch's version of "undefined behavior in numerical code" and is worth treating with the same seriousness SQLite treats UB in C.

## Suggested priority for future work

1. **PyTorch/NumPy cross-checking** — structurally different from everything done so far (MC/DC proves the existing tests are thorough; cross-checking proves the *values* are actually correct), and has a natural ground truth to compare against. Best next investment.
2. **Malformed state-file tests** — same bug family as two real bugs already found and fixed, likely has more waiting.
3. **Mutation testing** — cheap to run, would validate (or invalidate) how thorough the MC/DC work actually was, and surface anything missed.
4. Everything else on the table above, roughly in the order listed.
