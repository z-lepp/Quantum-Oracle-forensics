# Quantum Oracle Forensics

Solver for the **Quantum Oracle Forensics** challenge (IBM Qiskit Fall Fest 2026, QC@IITI × QryptoNox).

Given a sealed 8-bit oracle `U|x⟩|y⟩ = |x⟩|y ⊕ f(x)⟩` and a budget of 60 noisy queries,
`solve()` decides which of the eight families `f` belongs to and recovers its secret.

**Team:** Dheeraj VR, Faaz Muhammed, Himanshu Tembhurne, C Santosh

---

## Repository contents

| Path | What it is |
|---|---|
| `solver.py` | The submission: `solve(n, m, budget, families, run) -> (family, secret)` |
| `requirements.txt` | Dependencies of `solver.py` (Qiskit 2.x, numpy, scipy) |
| `qof_report.pdf` | Technical report, 3 pages (LaTeX source: `report/qof_report.tex`) |
| `logs/` | Logs of our own tests; every number in the report comes from these files |
| `circuit diagrams/` | Contains the circuit diagrams |
| `AI_usage_disclosure.pdf` | One-page AI-use disclosure |

---

## Setup

Python 3.10 or newer.

```bash
git clone https://github.com/z-lepp/Quantum-Oracle-forensics.git
cd Quantum-Oracle-forensics
python -m venv .venv
.venv\Scripts\activate            # Windows
# source .venv/bin/activate       # macOS / Linux
pip install -r requirements.txt
```

Tested with Python 3.13–3.14, Qiskit 2.5.2, numpy 2.x and scipy 1.18.

---

## Usage

```python
from solver import solve

family, secret = solve(n, m, budget, families, run)
```

| Argument | Meaning |
|---|---|
| `n` | number of input bits (8) |
| `m` | number of output bits (1 or 8) |
| `budget` | query budget (60) |
| `families` | list of family names the oracle may belong to |
| `run` | the judges' `run(circuit, shots)`: takes a Qiskit 2.x `QuantumCircuit`, returns a counts dictionary keyed by classical-bit strings in Qiskit order |

Every oracle call in the solver's circuits is the placeholder `Gate("oracle", n + m, [])`,
acting on the `n` input qubits followed by the `m` output qubits. `run()` replaces it with the
sealed oracle. The solver never reaches the oracle any other way.

### Return values

| Family | `m` | `secret` returned |
|---|---|---|
| `constant` | 1 | `c` as an int (0 or 1) |
| `balanced` | 1 | `None` |
| `linear` | 1 | `[a, b]`, ints, `a` in 1–255 |
| `grover` | 1 | `t` as an int (0–255) |
| `randfun` | 1 | `None` |
| `simon` | 8 | `s` as an int (1–255) |
| `simon2` | 8 | the three nonzero elements of `H`, ascending, as a list of ints |
| `perm` | 8 | `None` |

All values are plain Python `int`s, not numpy types.

---

## What `solve()` does at runtime

- Calls `run()` with **one shot per call**, at most 60 times per oracle, choosing each circuit based
  on the results so far.
- Counts its own queries (oracle gates × shots) and **never exceeds the budget**.
- Uses **at most 20 qubits** per circuit and puts **all measurements at the end**.
- Imports only **Qiskit, numpy, scipy** and the standard library. It reads no files, prints nothing
  and uses no network.
- Is **deterministic** for a given sequence of `run()` results (its own random choices use a fixed seed).
- Needs at most about 5 s of its own computation per oracle (measured up to 4.3 s on a 2-core
  machine, time spent inside `run()` excluded), far below the 120 s limit.

### Circuits used

| Circuit | Qubits | Queries per shot | Purpose |
|---|---|---|---|
| Deutsch–Jozsa / Bernstein–Vazirani | 17 | 1 | separates constant & grover / linear / balanced & randfun; gives `a` for linear |
| Grover echo (2 or 4 iterations, Hadamard readout) | 17 | 2 or 4 | separates balanced from randfun |
| Grover search (6–12 iterations) | 17 | 6–12 | finds the marked input `t` |
| Classical query of `f(x)` | 11 | 1 | reads `b`, `c`, and verifies a Grover candidate |
| Simon sampling (`m = 8`) | 20 | 1 | constraints on the hidden subspace (simon, simon2, perm) |

Spare qubits hold CNOT copies of the measured qubits (8 copies in the 17-qubit circuits, 2 in the
classical query, 4 in Simon). Readout flips are independent per measured bit, so copies that agree are
almost surely right, and a disagreement shows up as a detected error instead of a silent one. The
copies cost no queries.

---

## Strategy in brief

- **Exact Bayesian inference** over every possible answer: up to 1 + 255 + 10 795 hypotheses for
  `m = 8`, and constant / linear `(a, b)` / grover `t` / balanced / randfun for `m = 1`. The noise
  model (4% oracle misfire, 2% readout flip) is built into every likelihood.
- **The noise level is not assumed.** Each hypothesis is paired with both nominal and doubled noise,
  and the posterior averages over the two, so the stress setting needs no retuning.
- **Adaptive experiments.** Before each shot, the solver runs the circuit with the highest expected
  information gain per query.
- **Sequential stopping.** It answers as soon as one answer has posterior probability ≥ 1 − δ
  (δ = 0.003, chosen by a sweep in `logs/summary_delta_*.md`), or when the budget runs out.

The full description, with the decision rules, thresholds, budget split, baselines and failure
analysis, is in `report/qof_report.pdf`.

---

## Results from our own tests

400 random oracles, 50 per family, with the same oracles used in every experiment.

| Setting | Accuracy | Worst family | Query saving | Mean queries |
|---|---|---|---|---|
| Nominal noise (4%, 2%) | **0.998** | 0.98 | 0.591 | 24.5 |
| Stress noise (8%, 4%) | **0.990** | 0.96 | 0.463 | 32.2 |

Per-family accuracy and queries, the confusion matrices, the baselines (classical-only, single
algorithm, non-adaptive, random guess) and accuracy at budgets of 20–60 are in
`logs/summary_final.md` and the report.

### About the logs

The logs were produced with our own local test harness, which is not part of this repository. It
consists of:
- a NumPy state-vector simulator of the section-3 oracle and noise model (checked against every
  section-4 reference value and against Qiskit's `Statevector`);
- a benchmark script that runs `solve()` on fixed-seed random oracles.

| File | Contents |
|---|---|
| `logs/summary_final.md` | main results, baselines, budget sweep, confusion matrices, list of failures |
| `logs/bench_final.json` | one record per task: true family and secret, answer, queries used, time |
| `logs/summary_delta_*.md`, `logs/bench_delta_*.json` | stopping-threshold sweep, δ ∈ {0.03, 0.01, 0.003, 0.001}, 40 oracles per family |

Rows labelled `claude_reference` in the logs come from a separately written second solver. We used it
only to cross-check our results; it is not part of the submission.

---

## AI use

AI assistants were used during the hackathon. Details, and how the results were checked
independently, are in `AI_USE.md`.
