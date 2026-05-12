# CHANGELOG — MAPE-SVR fork

This fork of [LIBSVM](https://github.com/cjlin1/libsvm) adapts the
ε-SVR path to the MAPE (Mean Absolute Percentage Error) loss as
specified in Benavides-Herrera & Herrera-Espinoza (2026), "Sequential
Minimal Optimization for MAPE-SVR with Sample-Dependent Bounds and
Symmetric Kernel Extensions" (arXiv preprint `smo-v3.tex`,
Appendix A — *LIBSVM Drop-in Modification Recipe*).

Base: LIBSVM 3.37 (upstream commit `6b90713`).
Scope: ε-SVR (`-s 3`) only. C-SVC, ν-SVC, one-class, and ν-SVR are
unmodified.

---

## Modification 1 — Per-sample upper bound C_k

**Commit:** `feat(F10 Mod 1): per-sample C_k via get_C() setter override`
**Paper reference:** appendix A, lines 5352–5371
**Files:** `svm.cpp`

The scalar regularization bound `C` is replaced in ε-SVR mode by a
per-sample vector `C_k = 100·C/y_k`. The implementation adds an
optional `Q_alpha_bound` array member to the `Solver` class, set via
`Solver::set_Q_alpha_bound()`. When set, `Solver::get_C(i)` returns
`Q_alpha_bound[i]`; otherwise it falls back to scalar `Cp`/`Cn`
(LIBSVM-stock behavior). `solve_epsilon_svr()` allocates the array of
length `2*l`, populates both dual halves with the same per-sample
`C_k`, and passes it via the setter before invoking `Solver::Solve()`.

---

## Modifications 2, 3, 4 — automatically satisfied by LIBSVM 3.37's bound-check indirection

Appendix A Modifications 2 (working-set feasibility test), 3 (clipping
bounds in the two-variable update) and 4 (shrinking thresholds) are
specified in the paper as direct rewrites of comparisons like
`if (alpha[k] < C - 1e-8)` to `if (alpha[k] < Q_alpha_bound[k] - 1e-8)`.

In LIBSVM 3.37 such direct comparisons do not appear: all bound
inspection is routed through a single indirection layer. Once
Modification 1 (above) redirects `Solver::get_C()` to read from
`Q_alpha_bound`, Mods 2–4 inherit the per-sample bounds without any
further code edits.

The relevant call sites are:

| Mod | Function | svm.cpp lines | Why no edit is needed |
|----|----------|---------------|-----------------------|
| 2  | `Solver::select_working_set()` | 790–887 | Uses `is_upper_bound(i)` / `is_lower_bound(i)` (lines 442–444), which read `alpha_status[i]` cached by `update_alpha_status()` (lines 434–441), which calls `get_C(i)` (line 436). |
| 3  | `Solver::Solve()` two-variable update | 595–691 | Fetches `C_i = get_C(i); C_j = get_C(j);` once at lines 600–601 and reuses for all clipping decisions at lines 632–645, 659–681. |
| 4  | `Solver::do_shrinking()` + `Solver::be_shrunk()` | 889–907, 909–968 | Use only `is_upper_bound(i)` / `is_lower_bound(i)`. Same routing as Mod 2. |

The G_bar correction at `Solver::Solve()` initialization (line 558)
and after each two-variable step (lines 716, 719, 727, 730) also
routes through `get_C(i)` / the local `C_i`/`C_j`, and therefore
inherits the per-sample bounds correctly.

Per Theorem 1 of the paper (*structural invariance*), the kernel
evaluation (`Kernel::k_function()`, cache, column access), gradient
bookkeeping (`G[i]` array), KKT convergence check
(`Delta = G_max - G_min ≤ eps`), reconstruction/unshrinking, and bias
recovery all remain unchanged for MAPE-SVR.

---

## Modification 5 — Linear-coefficient vector for the eps-SVR dual

**Commit:** `feat(F10 Mod 5): MAPE linear_term in solve_epsilon_svr`
**Paper reference:** appendix A, lines 5440–5456
**Files:** `svm.cpp` (`solve_epsilon_svr()`, the in-loop initialization
of `linear_term[i]` and `linear_term[i+l]`)

Replaces

```cpp
linear_term[i]   = param->p - prob->y[i];   // standard eps-SVR
linear_term[i+l] = param->p + prob->y[i];
```

with

```cpp
linear_term[i]   = prob->y[i] * (param->p / 100.0 - 1.0);   // MAPE-SVR
linear_term[i+l] = prob->y[i] * (param->p / 100.0 + 1.0);
```

so that the dual's linear term matches the MAPE-SVR formulation (paper
equation 2.2): the `q[k]` coefficients become `y_k(ε/100 − 1)` and
`y_k(ε/100 + 1)` for the positive and negative slack variables
respectively.

### CLI-flag semantic change (`-p`)

After this patch the `-p <eps>` flag is interpreted as a **MAPE-tube
width in percentage points**, not an absolute tube width:

| Flag | Stock LIBSVM (eps-SVR) | This fork (MAPE-SVR) |
|------|------------------------|----------------------|
| `-p 5` | tube of ±5 in target units | tube of ±5% relative to each y_k |
| `-c 1` | scalar regularization C | C, scaled per-sample as 100·C/y_k internally |
| `-s 3` | eps-SVR | MAPE-SVR (patched) |
| `-s 4` | nu-SVR | unmodified (unpatched eps-SVR ν-formulation) |

The asymmetric shrinking-threshold pattern of Lemma 4 of the paper
emerges automatically from Mod 5 because
`τ_{N+k} = τ_k + 2·y_k·ε/100` follows from this `linear_term` change
(see Proposition on structural gap, paper §`sec:dual-kkt`); no
additional code edit is needed.

---

## Solver_NU::Solve and the Solver_NU path

`Solver_NU::Solve()` at `svm.cpp:1017–1023` delegates to
`Solver::Solve()` via inheritance, so any state set via
`set_Q_alpha_bound()` propagates through. However, ν-SVR
(`solve_nu_svr`) deliberately does *not* call the setter — the
appendix specifies ε-SVR only, and ν-SVR's `linear_term = ±prob->y[i]`
formulation would require a separate derivation outside paper scope.

---

## Build

```bash
make clean
make
```

Toolchain: any g++/clang. Verified on MSYS2 / MinGW-w64 (Windows
native build); the Linux makefile is the reference.

Produces:
- `svm-train` (or `svm-train.exe` on Windows)
- `svm-predict`
- `svm-scale` (unchanged from upstream)
- `libsvm.so.4` (or platform equivalent)

---

## Citation

If you use this fork, please cite the paper:

> Benavides-Herrera, P. & Herrera-Espinoza, R. (2026). Sequential
> Minimal Optimization for MAPE-SVR with Sample-Dependent Bounds and
> Symmetric Kernel Extensions. arXiv preprint.

The canonical R implementation of the algorithm is the `psvr` package
([Benavides-Herrera 2026](https://doi.org/...) — Zenodo DOI).
