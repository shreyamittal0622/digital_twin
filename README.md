# digital_twin

A Digital Twin is a computational model of a real physical system.Suppose you have a physical machine. You want to understand how it behaves, what errors it makes, and how it will perform under different conditions.Instead of testing every possible situation on the physical machine, you build a mathematical model that imitates its behaviour.You then compare the model's predictions with the actual machine's behaviour and improve the model whenever necessary.
The same idea can be applied to quantum computers.

System 1: The real quantum processor

This is the actual hardware, treated as a black box. You can submit quantum circuits to it and obtain measurement results, but you do not initially know all its internal error parameters.

System 2: The digital twin

This is a quantum simulator that you implement in Python.

It tries to reproduce the behaviour of the real processor by incorporating its imperfections, such as:
- Qubit relaxation.
- Dephasing.
- Readout errors.
- Imperfect quantum gates.
- Errors in two-qubit operations.
The objective is to make the simulated processor behave as closely as possible to the real processor.

A 3-qubit simulator is  used to track or mimic the behavior of three quantum bits.
##### Density matrix

A density matrix is used because a quantum system is not always in a single, perfectly known quantum state. It allows the simulator to represent both pure states and mixed states, which is especially important when modelling noise, relaxation, and decoherence.For a pure state, the density matrix is defined as $$\rho $$= $$|\psi\rangle\langle\psi|$$.
#####
1.The density matrix must equal its conjugate transpose.(Hermiticity)
2.The total probability must remain one.(Unit trace)
3.The density matrix cannot produce negative probabilities.

For a system containing \(n\) qubits, the Hilbert-space dimension is\[d=2^n.\]For three qubits,\[d=2^3=8.\]

For an ideal unitary gate \(U\), the state evolves according to $$\rho'$$=U $$\rho$$ $$U^\dagger$$

The three-qubit operator is $$U=H\otimes I\otimes I$$

###  The different types of noise
#### 1. Relaxation: $$\(T_1\)$$

Suppose a qubit is initially in the excited state: $$|1\rangle$$.Because of its interaction with the environment, it can lose energy and transition to the ground state: $$\|1\rangle\longrightarrow|0\rangle\$$ The characteristic timescale of this process is called the energy relaxation time, $$\(T_1\)$$ .

For an initially excited qubit, the excited-state population ideally follows
$$\boxed{P_1(t)=e^{-t/T_1}}$$
Equivalently, the probability of finding the qubit in the ground state is $$P_0(t)=1-e^{-t/T_1}$$

The general procedure is:
1. Prepare the qubit in \(|0\rangle\).
2. Apply an \(X\) gate to prepare \(|1\rangle\).
3. Wait for a time \(t\).
4. Measure the qubit.
5. Repeat for several delay times.
6. Fit the measured excited-state population to an exponential decay.

   Relaxation fitting model $$f_{T_1}(\tau)=1-2e^{-\tau/T_1}$$

Describes the Z expectation for an initially excited qubit relaxing toward the ground state.
#### 2. Dephasing: $$\(T_\phi\)$$

Relaxation is not the only way a qubit loses quantum information.

A qubit can also lose the relative phase between its basis states.

Consider
$$|\psi\rangle=\frac{|0\rangle+|1\rangle}{\sqrt2}.$$
Its density matrix is
$$\rho=\frac12\begin{pmatrix}1&1\\1&1\end{pmatrix}$$
Pure dephasing reduces the off-diagonal elements:
$$\rho(t)=\frac12
\begin{pmatrix}
1&e^{-t/T_\phi}\\
e^{-t/T_\phi}&1
\end{pmatrix}$$
Here, \(T_\phi\) is the pure-dephasing timescale.

How do we estimate dephasing?
The task specifies a Ramsey experiment. A typical Ramsey sequence is
$$\boxed{X_{\pi/2}\rightarrow\text{delay }t\rightarrow X_{\pi/2}\rightarrow\text{measurement}}$$

with appropriate phase choices for the pulses.

More generally, a Ramsey experiment uses two $$\(\pi/2\)$$ pulses separated by a period of free evolution.
The first pulse creates a superposition. During the delay, the relative phase evolves and becomes sensitive to frequency detuning and dephasing.

The second pulse converts phase information into a measurable population difference.

Coherence-time relationship $$\frac1{T_2}=\frac1{2T_1}+\frac1{T_\phi}$$

Relates energy relaxation and pure dephasing to the coherence time under the conventional model.

Ramsey-style fitting model $$f_R(\tau)=e^{-\tau/T_2^*}\cos(\Delta\tau)$$

#### 3. Han Echo
An initial radiofrequency pulse is applied to tip the net magnetization vector from the z-axis down into the xy-plane.

Combines an exponentially decaying envelope with an oscillatory signal.(The Preparation Pulse ($90^\circ$ or $\pi/2$ pulse))
The transmitter is turned off, allowing the spins to evolve freely for a set time interval ($\tau$) while local field variations cause them to dephase and the signal to decay. A second pulse (twice as long) is applied at time $\tau$ to instantly flip the orientation of all the dephasing spins by 180 degrees in the xy-plane. The system is left to evolve for another identical time interval ($\tau$), allowing the faster and slower spins to swap relative positions and catch up to each other.  Exactly at total time $2\tau$ (after the initial $90^\circ$ pulse), all the spins realign into a coherent bunch.

Hahn-echo fitting model $$f_E(\tau)=e^{-\tau/T_2}$$

Describes the exponential decay assumed for the simulated echo signal.

### Gate-amplitude errors
Quantum gates are implemented using physical control pulses.

For example, a rotation around the \(x\)-axis is
$$R_x(\theta)=e^{-i\theta X/2}$$
where \(X\) is the Pauli-\(X\) matrix.

Suppose you intend to apply
$$R_x\left(\frac{\pi}{2}\right)$$
The actual rotation might be
$$R_x\left((1+\eta)\frac{\pi}{2}\right),$$

The sequence performed is
$$R_x(\pi/2)[R_x(\pi)]^n.$$
The general idea is to apply a small calibration error repeatedly so that its effect becomes easier to observe.


#### What is Fisher information?
Fisher information measures how informative an experiment is about an unknown parameter.

Suppose the probability of measuring one is
$$p(t;T_1)=1-e^{-t/T_1}.$$
The probability depends on the unknown parameter $$\(T_1\)$$ .

If you choose a delay that makes the measurement outcome almost independent of $$\(T_1\)$$, the experiment provides little information about $$\(T_1\)$$.

If you choose a delay where the probability changes appreciably with $$\(T_1\)$$ , the measurement may be more informative.


### cross-validation
This is a crucial part of the project.

A model may fit its calibration data extremely well but still perform poorly on different circuits.

That can happen when the model is too simplistic, its parameters are poorly identified, or it has learned patterns specific to the calibration experiments.
What do you do?
You construct additional circuits with different structures, depths, or gate sequences.

The instructions mention examples such as:
- GHZ-like circuits.
- Random Clifford-like circuits.
These circuits test the model under different conditions.

#### How do you decide whether an additional effect is worth including?
You propose a physically justified modification to the model and refit the data.

Then compare the old and new models.

The instructions mention criteria such as AIC and BIC
The Akaike Information Criterion is
$$\boxed{\mathrm{AIC}=2k-2\log L_{\max}}$$ ,
where \(k\) is the number of fitted parameters and $$\(L_{\max}\)$$ is the maximum likelihood

The Bayesian Information Criterion is
$$\boxed{\mathrm{BIC}=k\log N-2\log L_{\max}}$$ ,
where \(N\) is the number of observations used in the relevant likelihood calculation, under the chosen convention.

Lower AIC or BIC generally indicates a better balance between fit quality and model complexity.

Adding parameters almost always gives the model more flexibility, but that does not automatically make it better.

The additional parameter should improve the explanation of the data enough to justify its complexity.


This is the final test of the digital twin before the hidden hardware parameters are revealed.

The instructions specify that you must predict the output probability distribution for every circuit in

heldout-circuits.json

without running those circuits on the black-box hardware.

This is called a blind prediction because you cannot use the actual results to adjust your predictions.

### Total Variance Distance
$$\mathrm{TVD}(\mathbf p,\mathbf q)=\frac12\sum_i|p_i-q_i|$$

Measures the disagreement between predicted and reference probability distributions.




