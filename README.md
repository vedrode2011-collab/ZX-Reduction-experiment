# ZX-Calculus Circuit Reduction: Which Reducer Is Better for Which Circuit?

Experiments comparing PyZX's circuit reduction strategies on different quantum circuits, checking that each reduction is correct, and measuring whether the "optimized" circuits actually give more accurate results on noisy (simulated and real) IBM hardware.

## Background

The ZX-calculus is a graphical language for quantum circuits. The [PyZX](https://github.com/zxcalc/pyzx) library implements a number of rewrite strategies that simplify a circuit by manipulating its ZX-diagram. This project compares two of them:

| Name in this repo | PyZX function | What it does | Extraction back to a circuit |
|---|---|---|---|
| **careful** | `zx.teleport_reduce` | Fuses and "teleports" matching phase gates through the circuit. The diagram stays circuit-like. | Direct: `zx.Circuit.from_graph()` |
| **aggressive** | `zx.full_reduce` | Full graph-like simplification (local complementation and pivoting) that minimizes spider count and T-count. The diagram is no longer circuit-like. | Needs `zx.extract_circuit()` (Gaussian elimination) |

The central question: **a reduced circuit looks simpler in the ZX-diagram, but does it run better on real hardware?** Extraction can add a lot of CNOTs, and on noisy devices more gates usually means more error.

## Circuits tested

| Notebook | Circuit | Size |
|---|---|---|
| `quantum_adder.ipynb` | Vedral-Barenco-Ekert ripple-carry adder (Toffoli/CNOT based) | `3n + 1` qubits for an `n`-bit adder (10 qubits for the 3-bit example) |
| `tfim_trotter.ipynb` | Trotterized transverse-field Ising model (TFIM), `J=1.0, g=0.5, dt=0.2, steps=3` | 4 qubits, scaled to 5 and 10 |

## Pipeline

Both notebooks follow the same steps:

1. **Build the baseline circuit** in Qiskit.
2. **Reduce it** with `teleport_reduce` (careful) and `full_reduce` (aggressive): Qiskit circuit → QASM → PyZX graph → reduce → extract → QASM → Qiskit circuit.
3. **Check equivalence** with `verify_equality` against the baseline. The function raises an error if a reduction changes the circuit's behavior, so every circuit that gets run is verified.
4. **Pre-flight metrics**: total gates, two-qubit gates, T-count, and depth for all three versions.
5. **Transpile** each version for the target backend's native gate set (optimization level 1).
6. **Run** on `FakeKingston` (noise model) and optionally on real `ibm_kingston` hardware, 4000 shots per circuit.
7. **Score accuracy** with the magnetization error metric below.

### Accuracy metric: magnetization error

An "all-zeros" success rate is misleading for circuits like the TFIM, where even a perfect machine does not return all zeros. Instead:

- **Measured magnetization**: for each shot, a qubit measured as `0` counts +1 and `1` counts -1, averaged over all qubits and shots.
- **Exact magnetization**: the same quantity from a noiseless statevector simulation (the answer key).
- **Error** = `|measured - exact|`. Lower is better.

All three circuit versions are verified equivalent, so they share one answer key.

## Results

### TFIM Trotter circuit (4 qubits)

Pre-flight metrics:

| Version | Total gates | Two-qubit gates | T-count | Depth |
|---|---|---|---|---|
| baseline | 39 | 18 | 21 | 24 |
| careful | 39 | 18 | 21 | 24 |
| aggressive | 89 | 36 | 21 | 61 |

Magnetization error (exact = 0.8645):

| Version | FakeKingston | Real `ibm_kingston` |
|---|---|---|
| baseline | 0.1764 | 0.2774 |
| careful | 0.1738 | 0.2876 |
| aggressive | 0.0873 | 0.0771 |

Error vs. qubit count (fake backend, 3 repetitions, mean ± std):

| Qubits | Exact | baseline | careful | aggressive |
|---|---|---|---|---|
| 5 | 0.8688 | 0.1482 ± 0.0014 | 0.1472 ± 0.0004 | 0.1863 ± 0.0080 |
| 10 | 0.8774 | 0.1143 ± 0.0040 | 0.1164 ± 0.0025 | 0.6012 ± 0.0478 |

Real-hardware job IDs (4 qubits, `ibm_kingston`): baseline `dani9678gn2s739mmcv0`, careful `dani97o2fm4c73f48ic0`, aggressive `dani99lr85ps73fe3dog`.

**Takeaways**

- `teleport_reduce` returned the circuit unchanged. The TFIM
