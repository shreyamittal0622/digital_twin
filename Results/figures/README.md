# Density-Matrix Digital Twin for Noisy Quantum Circuits

A 3-qubit density-matrix simulator that models gate errors, T1 relaxation, dephasing and readout error, then calibrates itself with synthetic experiments and propagates fit uncertainty into circuit predictions.

This README explains **every equation used in `notebook/digital_twin.ipynb`**, shows the notebook's results and graphs next to the equation that produced them, and lists the problems found along the way.

> **Important:** all data is synthetic. The simulator generates the traces and the same simulator family then fits them, so the results show *internal consistency*, not accuracy against real hardware.

---

## 1. Package contents

```
digital_twin_package/
├── README.md                  <- this file
├── requirements.txt           <- numpy, scipy, matplotlib, jupyter
├── notebook/
│   └── digital_twin.ipynb     <- the original notebook (digital_twin_1.ipynb was byte-identical)
├── report/
│   └── digital_twin_research_report.pdf
└── results/
    ├── figures/               <- 8 plots extracted from the notebook outputs
    ├── data/                  <- raw experiment traces (CSV)
    ├── metrics/               <- summary_metrics.json + all printed notebook output
    └── predictions/           <- blind predictions (median + 90% interval), mock circuits
```

**Run it:** `pip install -r requirements.txt`, then `jupyter notebook notebook/digital_twin.ipynb`. See [Section 8](#8-known-issues-and-fixes) first: the notebook only runs cleanly if you fix three cell-ordering problems.

---

## 2. Conventions

**Qubit ordering.** The state is an 8×8 matrix over the basis |q0 q1 q2⟩. The row/column index is

$$\text{index} = 4\,q_0 + 2\,q_1 + q_2$$

so applying X to qubit 0 moves the population from index 0 (|000⟩) to index 4 (|100⟩). The notebook confirms this (`rho[4,4] = 1`).

**Z sign convention.** ⟨Z⟩ = +1 for |0⟩ and −1 for |1⟩.

**Time units.** The simulator uses arbitrary time units (`au`). The notebook prints "µs", which is only meaningful if the time axis is calibrated that way.

**Simulator "true" parameters** used to generate all data:

| Parameter | Value |
|---|---|
| T1 | 50 |
| Tφ | 100 |
| T2 (implied, see Eq. 8) | 50 |
| Pulse over-rotation ε | 1° (0.01745 rad) |

---

## 3. State, gates and expectation values

### Eq. 1: initial state *(Cell 2)*

$$\rho_0 = |000\rangle\langle 000|,\qquad \rho_{0,0}=1,\ \text{all other entries }0$$

The density matrix ρ is an 8×8 Hermitian matrix with trace 1. Its diagonal holds the probabilities of the 8 basis states, and its off-diagonals hold the coherences (superposition information). Using ρ rather than a state vector is what lets the notebook represent *mixed* states produced by noise.

### Eq. 2: single-qubit gates *(Cell 2)*

$$I=\begin{pmatrix}1&0\\0&1\end{pmatrix},\quad X=\begin{pmatrix}0&1\\1&0\end{pmatrix},\quad Z=\begin{pmatrix}1&0\\0&-1\end{pmatrix},\quad H=\tfrac{1}{\sqrt2}\begin{pmatrix}1&1\\1&-1\end{pmatrix}$$

* **X** flips |0⟩ ↔ |1⟩.
* **Z** flips the sign of |1⟩ (a phase flip), and is also the observable measured.
* **H** (Hadamard) turns |0⟩ into (|0⟩+|1⟩)/√2, creating a superposition.

### Eq. 3: embedding a gate in 3 qubits *(`get_full_operator`, Cell 3)*

$$U_{\text{full}} = U_0 \otimes U_1 \otimes U_2,\qquad U_k = \begin{cases} g & k=q\\ I & k\neq q\end{cases}$$

A one-qubit gate g on qubit q is lifted to 8×8 using Kronecker products (⊗), with the identity on the other two qubits.

### Eq. 4: gate application *(`apply_single_gate`, Cell 3)*

$$\rho' = U_{\text{full}}\,\rho\,U_{\text{full}}^{\dagger}$$

The standard unitary update for a density matrix. It preserves the trace and the eigenvalues of ρ.

### Eq. 5: state validity *(`val_state`, Cell 4)*

$$\lambda_{\min}(\rho) \ge -10^{-12}$$

After each noise channel the code checks that no eigenvalue of ρ is meaningfully negative (positivity). **Caveat:** the function computes `Tr(ρ)` but never asserts it equals 1, so the notebook's "trace preserved" message is not actually tested (see Section 8).

### Eq. 6: Z expectation value *(`get_expectation_z`, Cell 5)*

$$\langle Z_q\rangle = \mathrm{Tr}\!\left(Z_q\,\rho\right),\qquad Z_q = I\otimes\cdots\otimes Z\otimes\cdots\otimes I$$

This is the "measurement" used in all calibration experiments. For one qubit it equals P(0) − P(1).

---

## 4. Noise model

### Eq. 7: amplitude damping (T1 relaxation) *(`apply_delay`, Cell 5)*

$$p_1(\tau)=1-e^{-\tau/T_1},\qquad K_0=\begin{pmatrix}1&0\\0&\sqrt{1-p_1}\end{pmatrix},\quad K_1=\begin{pmatrix}0&\sqrt{p_1}\\0&0\end{pmatrix}$$

$$\rho'=\sum_i K_i\,\rho\,K_i^{\dagger}$$

* **p₁** is the probability that an excited qubit decays to |0⟩ during a delay τ.
* **K₁** is the "jump" (|1⟩ → |0⟩). **K₀** is the "no jump" branch, which also shrinks the |1⟩ amplitude.
* The Kraus form Σ KρK† is how non-unitary (lossy) processes are written. The condition K₀†K₀ + K₁†K₁ = I guarantees the trace is preserved.
* Effect: |1⟩ population decays as e^(−τ/T1), and coherences shrink by √(1−p₁) = e^(−τ/2T1).

### Eq. 8: pure dephasing and the T2 relation *(`apply_delay`, Cell 5)*

$$p_\phi(\tau)=\tfrac12\left(1-e^{-\tau/T_\phi}\right),\qquad E_0=\sqrt{1-p_\phi}\,I,\quad E_1=\sqrt{p_\phi}\,Z$$

With probability p_φ a random Z (phase flip) is applied. This leaves populations alone and multiplies coherences (off-diagonals) by (1 − 2p_φ) = e^(−τ/Tφ).

**Combined coherence decay.** Multiplying the two effects:

$$e^{-\tau/2T_1}\cdot e^{-\tau/T_\phi}=e^{-\tau/T_2}\quad\Longrightarrow\quad \frac{1}{T_2}=\frac{1}{2T_1}+\frac{1}{T_\phi}$$

For T1 = 50 and Tφ = 100: 1/T2 = 1/100 + 1/100, so **T2 = 50**. I checked numerically that the simulator's Ramsey signal at τ = 30 equals e^(−30/50) = 0.5488 to 15 digits, so the phase-channel convention matches the intended T2 relation.

### Eq. 9: readout (confusion-matrix) error *(`measure_with_confusion`, Cells 6 and 10)*

$$\mathbf p_{\text{meas}} = M\,\mathbf p_{\text{ideal}},\qquad \mathbf p_{\text{ideal}}=\mathrm{diag}(\rho),\qquad M_{ij}\propto 0.95\,\delta_{ij}+0.005\,u_{ij}$$

M is an 8×8 matrix whose entry M_ij is the probability of reading outcome *i* when the true state is *j*. It is built as 0.95·I plus small random leakage u_ij ∈ [0,1) (seed 42), then each **column is normalized to sum to 1** so probabilities stay valid.

**Result:** after X on qubit 0 (true state index 4), the measured distribution still peaks at index 4 with probability **0.9805**.

---

## 5. Calibration experiments, models and fits

All four experiments sweep a parameter and record ⟨Z₀⟩. Each is paired with an analytic model that `scipy.optimize.curve_fit` fits to the simulated data. The 95% confidence half-width for every fit is

$$\text{CI}_{95}=1.96\,\sqrt{\Sigma_{\theta\theta}}$$

where Σ is the covariance matrix returned by the fit (*Cell 16*).

![Four calibration experiments](results/figures/01_four_experiments_overview.png)

*Top-left: T1 inversion recovery. Top-right: Ramsey. Bottom-left: Hahn echo. Bottom-right: pulse-error amplification.*

### 5.1 T1 inversion recovery *(Cells 7, 11, 16)*

**Sequence:** X (prepare |1⟩), delay τ, measure ⟨Z⟩. τ = 0…200, 30 points.

$$\langle Z\rangle(\tau)=1-2\,e^{-\tau/T_1}$$

The excited population is e^(−τ/T1), so ⟨Z⟩ = P(0) − P(1) = 1 − 2e^(−τ/T1). The signal climbs from −1 to +1.

**Validation (Cell 7):** at τ = T1 = 50 the simulator gives 0.2642, and 1 − 2e^(−1) = **0.2642**. The notebook compares against −e^(−1) = −0.3679, which is the wrong formula, so its "Analytic Target" line is the error, not the simulator.

| | Value |
|---|---|
| Fitted T1 | **50.00** ± 4.4×10⁻¹⁵ |

### 5.2 Ramsey fringes (T2\*) *(Cells 8, 12, 16)*

**Sequence:** H, delay τ, H, measure ⟨Z⟩. τ = 0…150, 40 points.

$$\langle Z\rangle(\tau)=e^{-\tau/T_2^{*}}\cos(\Delta\,\tau)$$

The first H puts the qubit on the equator; during the delay coherence decays (Eq. 8) and picks up a phase Δτ if the qubit is detuned; the second H converts the remaining coherence into a measurable population difference. This simulator has **no detuning**, so the cosine factor is ≈ 1 and the trace is a pure exponential.

| | Value |
|---|---|
| Fitted T2\* | **50.00** ± 1.1×10⁻⁸ |

### 5.3 Hahn echo (T2) *(Cells 13, 16)*

**Sequence:** H, delay τ/2, X (refocusing π-pulse), delay τ/2, H, measure. τ = 0…150, 40 points.

$$\langle Z\rangle(\tau)=e^{-\tau/T_2}$$

The middle X pulse swaps the |0⟩/|1⟩ branches so slow phase drift is undone. Here there is no slow drift to undo (only Markovian noise), so echo and Ramsey give the same T2.

| | Value |
|---|---|
| Fitted T2 (echo) | **50.00** ± 7.6×10⁻¹⁵ |

### 5.4 Pulse-amplitude (over-rotation) error *(Cells 14, 17)*

**Gate:** an X-rotation by angle θ:

$$R_x(\theta)=\begin{pmatrix}\cos\frac\theta2&-i\sin\frac\theta2\\-i\sin\frac\theta2&\cos\frac\theta2\end{pmatrix},\qquad \theta=\pi+\varepsilon,\ \ \varepsilon=1^\circ$$

Each "π pulse" actually rotates by π + ε. Repeating it makes the error pile up. The notebook applies **n + 1** pulses (one, then n more) for n = 0, 10, …, 290, and fits

$$\langle Z\rangle(n)=A\cos(n\,\varepsilon)+c$$

Starting from |0⟩, N faulty pulses give ⟨Z⟩ = cos(N(π+ε)) = (−1)^N cos(Nε). I verified this numerically against the simulator. Since N = n + 1 is odd for every n in the sweep, the true signal is **−cos((n+1)ε)**, whereas the fit model uses cos(nε). The mismatch of one pulse is why the fit returns 1.005° instead of exactly 1°, and why the residuals (Fig. 2d below) show a smooth structure instead of noise.

| | Value |
|---|---|
| True ε | 1.000° |
| Fitted ε | **1.005°** ± 0.00047° |

(The CI is far smaller than the 0.005° bias, which shows the CI measures fit precision only, not model correctness.)

### 5.5 Residuals *(Cell 21)*

$$r_i = y_i - f(x_i;\hat\theta)$$

Residuals are data minus fit. Flat noise around zero means a good model; smooth structure means model mismatch.

![T1 residuals](results/figures/02_residuals_T1.png)

*Fig. 2a: T1 residuals are ~10⁻¹⁶, i.e. floating-point rounding.*

![Ramsey residuals](results/figures/03_residuals_ramsey.png)

*Fig. 2b: Ramsey residuals reach ~10⁻¹⁰ with a smooth shape, probably because the extra detuning parameter lets the fit trade off slightly against T2\*. Still negligible.*

![Hahn echo residuals](results/figures/04_residuals_hahn_echo.png)

*Fig. 2c: Hahn echo residuals are ~10⁻¹⁶ (rounding).*

![Pulse error residuals](results/figures/05_residuals_amplitude_error.png)

*Fig. 2d: the pulse-error residuals are ~10⁻³ with clear structure. This is the one fit where the model is slightly wrong (the n versus n+1 mismatch above).*

### 5.6 SPAM-augmented T1 model *(Cell 22)*

$$\langle Z\rangle(\tau)=V\,\bigl(1-2e^{-\tau/T_1}\bigr)+b$$

Adds a **visibility** V (contrast lost to imperfect state preparation and measurement, SPAM) and an **offset** b (baseline shift).

![T1 with SPAM](results/figures/06_T1_fit_with_SPAM_model.png)

| Parameter | Fitted |
|---|---|
| T1 | 50.0000 |
| Visibility V | 1.0000 |
| Offset b | ≈ 0 (−1×10⁻¹¹) |

Because the synthetic data has no SPAM error, V = 1 and b = 0 are the correct answers. SciPy emitted `OptimizeWarning: Covariance of the parameters could not be estimated`, so no confidence intervals exist for this fit.

### Eq. 10: model comparison with AIC *(Cell 23)*

$$\mathrm{AIC}=n\ln\!\left(\frac{\mathrm{RSS}}{n}\right)+2k,\qquad \mathrm{RSS}=\sum_i r_i^2$$

n = number of points, k = number of parameters; **lower is better**. The first term rewards fit quality, the second penalizes extra parameters.

| Model | k | AIC |
|---|---|---|
| T1 only | 1 | **−2198.02** |
| T1 + SPAM | 3 | −1564.09 |

The simpler model wins, which is the correct conclusion for data with no SPAM error. Note, however, that both RSS values are at floating-point-rounding level (roughly 10⁻³¹ versus 10⁻²², back-computed from the AIC), so the size of the gap reflects rounding, not statistical evidence. With realistic measurement noise the AIC difference would be far smaller.

---

## 6. Multi-qubit gates and circuit examples

### Eq. 11: CNOT embedding *(`apply_cnot`, Cell 18)*

$$\mathrm{CNOT}=\begin{pmatrix}1&0&0&0\\0&1&0&0\\0&0&0&1\\0&0&1&0\end{pmatrix}$$

Flips the target qubit when the control is |1⟩. For (control 0, target 1) it is lifted as CNOT ⊗ I. For (control 0, target 2) the notebook builds the 8×8 permutation matrix bit by bit:

$$P_{f(i),\,i}=1,\qquad f(i)=\text{index of } i \text{ with bit}_{\text{target}}\text{ flipped if bit}_{\text{control}}=1$$

**Caveat:** any other control/target combination silently returns the identity (does nothing).

### GHZ-style test *(Cell 19)*

Circuit: H on qubit 0, then CNOT(0 → 2):

$$|000\rangle\ \xrightarrow{H_0}\ \tfrac{1}{\sqrt2}(|000\rangle+|100\rangle)\ \xrightarrow{\text{CNOT}_{0\to2}}\ \tfrac{1}{\sqrt2}(|000\rangle+|101\rangle)$$

**Result:** ⟨Z₀⟩ = **0.0**, as expected for an equal superposition. This is a two-qubit Bell-type state (qubit 1 is untouched), not a full three-qubit GHZ state.

### Random circuit *(`run_random_circuit`, Cell 20)*

Each layer applies a random gate (H or X) to a random qubit, then a delay of 10 on that qubit. Results: 2 layers ⟨Z₀⟩ = **−0.6375**; 10 layers ⟨Z₀⟩ = **−0.4165**. The notebook uses Python's unseeded `random` module, so these are one run, not benchmarks.

---

## 7. Uncertainty propagation and sensitivity

### Eq. 12: Monte Carlo parameter sampling *(Cell 25)*

$$\boldsymbol\theta^{(s)}=(T_1,\ T_2^{*},\ \varepsilon)^{(s)}\sim\mathcal N\!\left(\hat{\boldsymbol\theta},\ \mathrm{diag}(\Sigma_{11},\Sigma_{22},\Sigma_{33})\right),\quad s=1,\dots,200$$

Each of 200 parameter sets is drawn from a normal distribution centred on the fitted values, using only the *diagonal* of the fit covariance (parameters treated as independent). The circuit is simulated once per sample, giving 200 probability vectors p⁽ˢ⁾ = diag(ρ⁽ˢ⁾).

### Eq. 13: prediction and 90% interval *(Cell 25)*

$$\hat p_k=\mathrm{median}_s\,p_k^{(s)},\qquad \bigl[\,P_5\!\left(p_k^{(s)}\right),\ P_{95}\!\left(p_k^{(s)}\right)\bigr]$$

For each of the 8 basis states, the median is the prediction and the 5th/95th percentiles form the 90% interval.

**Results**

| Basis state (q0 q1 q2) | Circuit 001 (H₀, delay 50 on all, faulty Rx(π) on q1) | Circuit 002 (X₂, delay 100 on all) |
|---|---|---|
| 000 | 0.0001 | **0.8647** |
| 001 | 0 | 0.1353 |
| 010 | **0.8160** | 0 |
| 110 | 0.1839 | 0 |
| others | 0 | 0 |

These match hand calculations. In circuit 002 the excited qubit survives with probability e^(−100/50) = 0.1353, giving 0.8647 in |000⟩ and 0.1353 in |001⟩. In circuit 001, after H the qubit-0 excited population is ½ and relaxes to ½·e^(−1) = 0.1839 at τ = T1, and the qubit-1 over-rotation of 1° leaves it almost fully in |1⟩.

![Blind prediction](results/figures/07_blind_prediction_circuit_001_90pct_intervals.png)

*Fig. 4: predicted probabilities for circuit 001. The error bars are invisible because the interval width is essentially zero (the fit covariance is ~10⁻³⁰ since the data is noise-free). The x-axis labels are also garbled; see Section 8.*

### Eq. 14: total variation distance *(`calculate_tvd`, Cell 27)*

$$\mathrm{TVD}(p,q)=\tfrac12\sum_{k=1}^{8}\left|p_k-q_k\right|$$

A distance between two probability distributions: 0 = identical, 1 = completely disjoint. It is the largest possible difference in the probability of any event.

### Eq. 15: finite-difference sensitivity *(Cells 29 and 30)*

$$\frac{\partial\,\mathrm{TVD}}{\partial\theta}\approx\frac{\mathrm{TVD}(\theta+\delta)-\mathrm{TVD}(\theta-\delta)}{2\delta}$$

A central finite difference measures how quickly the TVD changes when a parameter is nudged by ±δ (δ = 0.5).

### Eq. 16: error attribution *(Cell 30)*

$$\Delta_\theta=\frac{\partial\,\mathrm{TVD}}{\partial\theta}\cdot\bigl(\hat\theta-\theta_{\text{true}}\bigr)$$

First-order estimate of how much of the TVD error comes from the parameter error (estimate − truth).

![TVD breakdown](results/figures/08_posthoc_TVD_error_breakdown.png)

| Parameter | Estimated | "True" used | Contribution Δ |
|---|---|---|---|
| T1 | 50.00 | 40 (placeholder) | 2.83×10⁻⁶ |
| T2\* | 50.00 | 35 (placeholder) | 3.3×10⁻¹⁵ |
| Gate error ε | 0.01753 rad | 0.01 (placeholder) | −5.18×10⁻⁵ |

**This chart is a demonstration of the workflow only.** The "true" values are dummies, and the reference distribution is `[1,0,0,0,0,0,0,0]` instead of a measured one. Treat the bar heights as meaningless until real references are supplied.

---

## 8. Known issues and fixes

Items 1 to 3 stop the notebook running top-to-bottom. The rest affect interpretation.

| # | Issue | Fix |
|---|---|---|
| 1 | `plt` is used in Cell 15 but imported in Cell 21 | Add `import matplotlib.pyplot as plt` to Cell 1 |
| 2 | `blind_predictions` is used in Cell 25 but never created | Add `blind_predictions = {}` before the loop |
| 3 | `delta` is used in Cell 29 but only defined in Cell 30 | Move `delta = 0.5` above Cell 29 |
| 4 | T1 validation (Cell 7) compares against −e^(−τ/T1) | Use 1 − 2e^(−τ/T1) = 0.2642 |
| 5 | `val_state` never checks Tr(ρ) = 1 | Add `assert abs(np.trace(rho)-1) < 1e-10` |
| 6 | Pulse-train fit model uses cos(nε), but the simulation applies n+1 pulses and the true signal is −cos((n+1)ε) | Fit −cos((n+1)ε) or fit with `n+1`; this removes the 0.005° bias |
| 7 | Cell 25 passes the fitted **T2\*** into `apply_delay` as the **Tφ** argument (`apply_delay(rho, tau, t1_sample, t2_sample, q)`). The same happens in the TVD function (Cell 27) | Convert first: 1/Tφ = 1/T2\* − 1/(2T1). Currently harmless, because both example circuits are measured in the Z basis and dephasing doesn't change populations, but it would give wrong answers for circuits whose result depends on coherence |
| 8 | Monte Carlo intervals have zero width (covariance ≈ 10⁻³⁰) | Add realistic shot noise before fitting, or inflate/bound the covariance; use correlated, truncated sampling |
| 9 | Plot labels in Cell 26 render as "\|1003\>" etc. | Use `f"\|{i:03b}>"` |
| 10 | Sensitivity step δ = 0.5 is also applied to ε, which is in radians (0.5 rad ≈ 28.6°) | Use a separate small step for ε (for example 10⁻³ rad) |
| 11 | CNOT falls back to identity for unsupported qubit pairs | Implement the general bit-permutation for all pairs |
| 12 | Cell 14 comment says "0, 2, 4 … 18 repetitions", but `range(0, 300, 10)` runs 0…290 | Correct the comment |
| 13 | Random-circuit results are unseeded | Call `random.seed(...)` |
| 14 | Printed units say µs but the simulation is in arbitrary units | Label axes "au" or calibrate |

---

## 9. Results at a glance

| Quantity | True value | Fitted | 95% CI half-width |
|---|---|---|---|
| T1 | 50 | 50.00 | 4.4×10⁻¹⁵ |
| T2\* (Ramsey) | 50 | 50.00 | 1.1×10⁻⁸ |
| T2 (Hahn echo) | 50 | 50.00 | 7.6×10⁻¹⁵ |
| Pulse error ε | 1.000° | 1.005° | 0.00047° |

| Check | Result |
|---|---|
| Initial state | 8×8, ρ₀₀ = 1 |
| X on qubit 0 | population at index [4,4] |
| T1 test at τ = T1 | 0.2642 (correct analytic value 0.2642) |
| Readout peak | index 4, 0.9805 |
| GHZ-style ⟨Z₀⟩ | 0.0 |
| AIC (T1 / T1 + SPAM) | −2198.02 / −1564.09 |

**Where each result lives:** figures in `results/figures/`, raw traces in `results/data/`, fitted numbers in `results/metrics/summary_metrics.json`, predictions in `results/predictions/`.

**Honest summary:** the simulator is correctly built (the physics checks out analytically), the calibration pipeline recovers the programmed parameters, and the uncertainty and sensitivity stages are working demonstrations. Nothing here yet shows accuracy against a real device. To get there, add measurement noise, replace the placeholder truth values, fix the issues in Section 8, and compare against independently generated or hardware data.
