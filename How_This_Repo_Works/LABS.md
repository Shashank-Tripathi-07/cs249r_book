# Co-Labs -- Deep Dive

> **Purpose:** Explain how the 33 interactive labs work, what MLSys-im is and why it matters,
> how the browser execution model works, and the Design Ledger persistence system.
> Read REPO_OVERVIEW.md first.

---

## What Are Co-Labs?

Co-Labs sit between the textbook (read) and TinyTorch (build). They're the "try it and
discover" layer. Every lab gives students a prediction problem -- "guess what happens when
you double the batch size" -- then shows them the real answer through an interactive
instrument. The gap between prediction and reality is the intended learning moment.

33 labs total, split into two volumes:
- **Volume I (17 labs):** Single-machine ML systems -- one GPU, one training run
- **Volume II (16 labs):** Distributed systems, fleets, scale, real-world operations

---

## The Tech Stack

```
Student's browser
  |
  |-- Marimo notebook (.py files in labs/vol1/ and labs/vol2/)
  |     A Python notebook format that compiles to reactive JavaScript.
  |     Each cell is a function. Marimo tracks dependencies and re-runs
  |     downstream cells when an upstream cell changes.
  |
  |-- Pyodide (WebAssembly Python runtime)
  |     The full CPython interpreter, compiled to WASM, running in the browser.
  |     This is why no server is needed -- Python runs client-side.
  |
  |-- mlsysim (compiled into the WASM bundle)
        The hardware simulation library. Labs call it to get real hardware specs
        and run physics-grounded calculations.
```

**Why WASM?** Two reasons: (1) no install barrier, students just click a URL and it works;
(2) no server costs -- once the page loads, everything runs locally in the browser.

The tradeoff: WASM startup is slow (~5-10 seconds for the Python runtime to boot).
After that, individual cell re-runs are fast.

---

## Lab File Structure

Each lab is a single `.py` file:

```
labs/
|-- vol1/
|   |-- lab_00_introduction.py
|   |-- lab_01_ml_intro.py
|   |-- lab_02_ml_systems.py
|   ... (17 total)
|-- vol2/
|   |-- lab_01_introduction.py
|   |-- lab_02_compute_infra.py
|   ... (16 total)
|-- assets/            CSS, scripts, images shared across labs
|-- tests/             Static validation tests (structure, not execution)
|-- PROTOCOL.md        Pointer to .claude/docs/labs/PROTOCOL.md (the spec)
|-- TEMPLATE.md        Template and quality checklist for new labs
```

Despite being `.py` files, they're Marimo notebooks -- the `app = marimo.App()` pattern
at the top is Marimo's format. Don't confuse with regular Python scripts.

---

## Inside a Lab: The Zone Structure

Every lab follows the same architectural pattern (enforced by PROTOCOL.md):

```
ZONE A: OPENING
  Cell 0: Setup     -- imports, WASM package installs, hardware constants from mlsysim
  Cell 1: Briefing  -- learning objectives, the core question for this lab

ZONE B: PARTS (repeated 5 times, once per Part A through E)
  Prediction cell   -- radio buttons or numeric input, locked before exploring
  Instrument cells  -- sliders, charts, tables -- the interactive part
  Reveal cell       -- shows whether prediction was right, explains why

ZONE C: SYNTHESIS
  Takeaways cell    -- 3 key lessons from this lab
  Connections cell  -- links back to textbook section, links forward to next lab
  Design Ledger     -- saves the student's decisions for future labs to read
```

The prediction-lock pattern is deliberate: students commit to a prediction before they can
interact with the instrument. This prevents the "I would have guessed that" cognitive bias.

---

## MLSys-im: The Simulation Library

Labs don't hardcode hardware numbers. They call `mlsysim`, a physics-grounded library
that models everything from a Raspberry Pi to an H100 cluster.

### What mlsysim Provides

```python
# Get real hardware specs
H100_TFLOPS = mlsysim.Hardware.Cloud.H100.compute.peak_flops.m_as("TFLOPs/s")
iPhone_BW   = mlsysim.Hardware.Mobile.iPhone15Pro.memory.bandwidth.m_as("GB/s")

# Model a workload
from mlsysim.models import Llama3_70B
flops = Llama3_70B.total_flops(batch_size=32)

# Run a physics-based performance estimate
from mlsysim.core.solver import solve
result = solve(model=Llama3_70B, hardware=H100, batch_size=32)
print(result.latency_ms, result.throughput_tps)
```

### The 5-Layer Architecture

mlsysim separates what you're running from what you're running it on:

```
Layer A  Workload Representation  (mlsysim.models)
         FLOPs, parameters, arithmetic intensity
         Examples: Llama3_8B, ResNet50, GPT2

Layer B  Hardware Registry        (mlsysim.hardware)
         Real silicon specs: compute, memory, power
         Examples: H100, TPUv5p, Jetson Nano, iPhone15Pro, RPi4

Layer C  Infrastructure           (mlsysim.infra)
         Datacenter: PUE (power overhead), carbon intensity, WUE (water)

Layer D  Systems & Topology       (mlsysim.systems)
         Fleet configs, network fabrics between nodes
         Examples: multi-GPU clusters, edge deployment scenarios

Layer E  Execution Engine         (mlsysim.core.solver)
         The math: roofline model, memory-bound vs compute-bound analysis,
         design space search (optimizer)
```

### Why labs use mlsysim instead of hardcoded numbers

If H100 specs change (they do, via firmware and driver updates), a hardcoded number
in a lab file would silently become stale. With mlsysim, updating the hardware registry
file propagates everywhere automatically. Labs also inherit unit handling -- no bug where
someone forgot to convert GB/s to bytes/ns.

### The WASM Delivery

When a lab loads in the browser:
1. Marimo's JS runtime starts
2. Pyodide boots (this is the slow part)
3. `micropip.install("../../wheels/mlsysim-0.1.0-py3-none-any.whl")` runs
4. The `.whl` file is a pre-built wheel file bundled in the repo at `labs/assets/wheels/`
5. After install, `import mlsysim` works normally

The platform check `if sys.platform == "emscripten":` at the top of every lab handles
the two execution modes: browser (WASM, uses micropip) vs local (normal Python, uses
the installed package from the monorepo).

---

## The Design Ledger

This is the feature that makes labs cumulative rather than isolated.

Every prediction and design decision a student makes gets saved to their browser's
localStorage as a structured JSON object via `mlsysim.labs.state.DesignLedger`.

Later labs can read decisions from earlier labs:
- Lab 08 (Training Memory Budget) reads Lab 05's activation function choice
- Lab 11 (Roofline) reads the hardware choice from Lab 02
- Volume II capstone (Lab 16) reads all decisions from all previous labs

This creates a personal "portfolio" of design choices across the course.

**Important limitation:** localStorage is browser-local. If a student clears their browser
data, switches browsers, or uses a different device, the Design Ledger resets. There's no
server-side persistence for student data.

---

## Lab Inventory

### Volume I: Foundations (17 labs)

Single-machine ML systems -- these labs assume one GPU or CPU running one workload.

```
00  lab_00_introduction     The Architect's Portal -- orientation, how labs work
01  lab_01_ml_intro         The Magnitude Awakening -- orders of magnitude in ML
02  lab_02_ml_systems       The Iron Law -- Amdahl's Law, scaling limits
03  lab_03_ml_workflow      The Silent Degradation Loop -- training instability
04  lab_04_data_engr        The Data Gravity Trap -- data movement costs
05  lab_05_nn_compute       The Activation Tax -- activation function compute cost
06  lab_06_nn_arch          The Quadratic Wall -- attention's O(n^2) complexity
07  lab_07_ml_frameworks    The Kernel Fusion Dividend -- framework optimization
08  lab_08_model_train      The Training Memory Budget -- forward/backward memory
09  lab_09_data_selection   The Data Selection Tradeoff -- quality vs quantity
10  lab_10_model_compress   The Compression Frontier -- quantization/pruning tradeoffs
11  lab_11_hw_accel         The Roofline -- compute-bound vs memory-bound analysis
12  lab_12_perf_bench       The Speedup Ceiling -- Amdahl again, practical limits
13  lab_13_model_serving    The Tail Latency Trap -- P99 vs median latency
14  lab_14_ml_ops           The Silent Degradation Problem -- prod model drift
15  lab_15_responsible_engr There Is No Free Fairness -- accuracy-fairness tradeoff
16  lab_16_ml_conclusion    The Architect's Audit -- capstone synthesizing vol1
```

### Volume II: At Scale (16 labs)

Distributed ML systems -- multi-node, multi-GPU, datacenter-scale problems.

```
01  lab_01_introduction     The Scale Illusion -- single machine vs cluster thinking
02  lab_02_compute_infra    The Compute Infrastructure Wall -- cluster architecture
03  lab_03_communication    Communication at Scale -- interconnect bandwidth limits
04  lab_04_data_storage     The Data Pipeline Wall -- distributed data loading
05  lab_05_dist_train       The Parallelism Puzzle -- data/model/pipeline parallelism
06  lab_06_fault_tolerance  When Failure Is Routine -- MTBF at scale
07  lab_07_fleet_orch       The Scheduling Trap -- job scheduling, preemption, queuing
08  lab_08_inference        The Inference Economy -- serving vs training compute modes
09  lab_09_perf_engineering The Optimization Trap -- when to profile, when to optimize
10  lab_10_edge_intelligence The Edge Thermodynamics Lab -- power and thermal limits
11  lab_11_ops_scale        The Silent Fleet -- monitoring at scale
12  lab_12_security_privacy The Price of Privacy -- differential privacy costs
13  lab_13_robust_ai        The Robustness Budget -- adversarial vs natural shift
14  lab_14_sustainable_ai   The Carbon Budget -- training carbon footprint
15  lab_15_responsible_ai   The Fairness Budget -- fairness at scale
16  lab_16_fleet_synthesis  The Fleet Synthesis -- vol2 capstone
```

---

## Testing Labs

Labs have static tests that run in CI without actually executing the notebooks:

```bash
# From labs/ directory
pytest tests/test_static.py -v
```

These tests check:
- Every lab file has the required Zone structure (A, B, C cells)
- Every lab imports mlsysim (not hardcoded numbers)
- Prediction cells exist before instrument cells in each Part
- Design Ledger writes are present in synthesis zone

There's no execution test in CI -- running 33 WASM notebooks in CI would be too slow and
requires a browser environment. Functional testing is manual.

---

## How to Add a New Lab

1. Copy `labs/TEMPLATE.md` and read it -- it has the full cell-by-cell spec
2. Check `labs/.claude/docs/labs/PROTOCOL.md` for the authoritative quality checklist
3. Create `labs/vol1/lab_NN_slug.py` (or vol2) following the Zone A/B/C structure
4. Add the lab to `labs/_quarto.yml` nav tree
5. Make sure it imports from mlsysim (no hardcoded hardware numbers)
6. Run `pytest tests/test_static.py` -- your lab will fail if structure is wrong
7. Test manually with `marimo run labs/vol1/lab_NN_slug.py`

---

## Common Gotchas

**1. Slow first load**
WASM startup takes 5-10s. Students think it's broken. The loading spinner needs to be
prominent. Don't report this as a bug unless startup takes >30s.

**2. Platform check is required**
The `if sys.platform == "emscripten"` block must be in every lab. Without it:
- In browser: mlsysim fails to import (no micropip install ran)
- Locally: micropip doesn't exist, throws ImportError

**3. Design Ledger is not synced**
localStorage is per-browser. No server stores it. Students who switch machines lose progress.
This is a known limitation, not a bug.

**4. WASM wheels must be pre-built**
The `mlsysim-*.whl` in `labs/assets/wheels/` is a compiled wheel. When mlsysim gets a
new version, the wheel must be rebuilt and committed. Labs won't update to a new mlsysim
version automatically.
