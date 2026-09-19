# Literature Review

> **Reading note.** Positions in the tables below are the author's reading of each architecture and should be treated as an interpretation, not a published taxonomy. The 2025–2026 entries were read at abstract level in a single literature pass (29 July 2026, re-run 4 August 2026). Every method detail, number, author list and venue must be confirmed against the source before it is cited in a submission or repeated in conversation.

## 1. Where the robot body lives — the axis that separates this work

The distinguishing axis is **not** the sensing modality (force sensing is now saturated in the literature). It is *where the robot body lives*. In the learned families the body lives in the weights, so a new arm costs target-robot data and gradient steps. Here the body lives in code below a written specification, so a new arm costs engineering time. Neither cost is obviously smaller, and this project does not argue that it is — but the engineered cost is auditable, needs no data pipeline, and lets a safety argument made once survive the port.

| Family | Representative work | Where force enters | Where the robot body lives | What it shows | Gap relative to this project |
|---|---|---|---|---|---|
| Vision-language-action generalists | Open X-Embodiment / RT-X [13] | Absent or late-fused | In the weights, via per-embodiment tokenizers | Language + vision generalisation across many robots, trained on pooled multi-robot data | No force regulation; cross-embodiment achieved by data volume, not by an interface. A new body needs data. |
| Force-augmented VLA | ForceVLA [2], ForceVLA2 [3] | Observation, and increasingly the action | In the weights | Fusing a wrench stream into a VLA improves insertion success over baselines; ForceVLA2 places a force/position split inside the learned model | Force and motion split is *learned*, not written; the body remains inside the weights, so porting needs retraining. |
| Slow-fast visual-force imitation | Reactive Diffusion Policy [5], ImplicitRDP [6], PhaForce [7], Force Policy [4] | Both loops, scheduled by contact phase | In the weights of *both* loops | Slow-fast timescale separation works; PhaForce reports that removing the fast layer collapses out-of-distribution performance. Force Policy recovers a local interaction frame from demonstrations (close in spirit to the task frame used here) | The fast corrector is learned on one arm's mass/friction/servo bandwidth, so it does not port and carries no bound. Timescale separation itself is settled — it is *not* claimed here. |
| Control-centric force RL | Beltran-Hernandez et al. [14], Martin-Martin et al. [15] | The controller regulates it; the policy shapes targets or gains | Partly in the controller, partly in learned gains | Force control can be learned on a position-controlled arm (UR3e). Closest academic ancestor. | Action vector carries controller gains, so the policy may retune the robot; no portability study reported. Here the action carries **no gains** and force/motion are hard-partitioned. |
| Cross-embodiment transfer | Mirage [8], KITE [9], Contact-Anchored Policies [10], Van den Bogert et al. [11], XMoP [12] | Usually absent | Isolated by image inpainting, IK retargeting, or a learned latent seam | Zero-shot transfer of vision policies (Mirage); one frozen checkpoint across several arms for pick/open/close with no force sensing (CAP); frozen motion-planning policy across seven manipulators with no contact (XMoP); tactile-shear substitution for a compliant co-manipulation policy (Van den Bogert) | None regulates a **commanded normal force to a set point** through a **frozen policy and a written numeric contract** onto a **kinematically different position/velocity-commanded arm** with zero target data. Dropping any one of those three qualifiers admits a counterexample above. |
| Classical hybrid force/motion control | Raibert & Craig [16], Mason [17], Hogan [18] | The controller, by axis partition | Entirely in the controller | Force and position controlled on orthogonal axes in a task frame; impedance/admittance relations between force and motion | Analytic only — no learned search, alignment or phase logic. This project keeps the axis partition as the organising principle of the *action vector* and puts a learned policy above it. |
| Sim-to-real for contact | Zhang et al. [20], safe contact-rich RL survey [21], domain randomisation [22] | Measured wrench closes an inner loop | In the adapter (residual admittance), or in the randomisation ranges | Online admittance-residual adaptation below the policy narrows the sim-to-real contact gap; the contact gap is the classic failure point for this class | Used here as *tools*, not contributions. Residual adaptation lives strictly below the contract and every portability number is reported with and without it. |
| **This project** | This work | **Primary input** and the **commanded action on the normal axis**, regulated analytically at 1 kHz | **Entirely below a written contract**; no embodiment information reaches the policy | (To be measured — see `05_expected_results.md`) | — |

## 2. Supporting methods used below the contract (not contributions)

| Topic | Reference | Role in this project |
|---|---|---|
| Damped least-squares (singularity-robust) inverse kinematics | Nakamura & Hanafusa [23], Wampler [24], Chiaverini et al. [25], Maciejewski & Klein [26] | The adapter's 1 kHz differential IK is written from scratch on this method (finite-difference Jacobian, LDLᵀ solve of `JJᵀ + λ²I`). |
| Continuous 6D rotation representation | Zhou et al. [19] | Justifies observing end-effector orientation as the first two columns of the measured rotation matrix. Observation side only — no rotation is ever reconstructed, so no orthogonalisation step exists in the loop. |
| Soft actor-critic | Haarnoja et al. [27] | Training algorithm for the frozen policy in MuJoCo. |
| Cost/safety review of contact-rich RL | Parnada et al. [1] | Framing for why RL is not yet industrially viable in contact-rich work. |

## 3. Summary of the gap

Two literature passes on two days located three families of near neighbours and **no direct match** for: *transfer of a skill that regulates a commanded normal force to a set point, through a frozen policy and a written numeric contract, onto a kinematically different position- or velocity-commanded arm, with zero target-robot data and zero gradient steps on the policy.* This is an absence of evidence, stated in the narrowest defensible form, not a proof of absence.

---

## References

All entries below are **[UNVERIFIED]** in the sense required by this project: bibliographic details are taken from public listings or from abstract-level reads, and each must be checked against the primary source before it is cited in a submission. Entries marked † were deliberately recorded without full detail and must be completed from the source.

1. A. Parnada et al., "Towards cost-effective and safe contact-rich robotic manipulation with reinforcement learning: a review," 2026.
2. J. Yu et al., "ForceVLA: enhancing vision-language-action models with a force-aware mixture of experts for contact-rich manipulation," NeurIPS 2025. arXiv:2505.22159.
3. Li et al., "ForceVLA2: hybrid force-position control with force awareness," arXiv:2603.15169, 2026.
4. Fang et al., "Force Policy: hybrid force-position control under an interaction frame," arXiv:2602.22088, 2026.
5. H. Xue et al., "Reactive Diffusion Policy: slow-fast visual-tactile policy learning," arXiv:2503.02881, 2025.
6. Chen et al., "ImplicitRDP: end-to-end visual-force diffusion with structural slow-fast learning," arXiv:2512.10946, 2025.
7. Wang et al., "PhaForce: phase-scheduled visual-force policy with slow planning and fast correction," arXiv:2603.08342, 2026.
8. L. Y. Chen et al., "Mirage: cross-embodiment zero-shot policy transfer with cross-painting," RSS 2024. arXiv:2402.19249.
9. "KITE: kinematic interaction transfer across embodiments," arXiv:2606.22113, 2026.
10. "Contact-Anchored Policies," arXiv:2602.09017, 2026.
11. † Van den Bogert et al., cross-embodiment transfer of a compliant contact policy using tactile shear substitution, 2026. Title, author list and venue unconfirmed.
12. † XMoP, transfer of a frozen whole-body motion-planning policy across commercial manipulators, 2026. Title, author list and venue unconfirmed.
13. Open X-Embodiment Collaboration, "Open X-Embodiment: robotic learning datasets and RT-X models," 2023.
14. C. C. Beltran-Hernandez et al., "Learning force control for contact-rich manipulation tasks with rigid position-controlled robots," IEEE Robotics and Automation Letters, vol. 5, no. 4, pp. 5709–5716, 2020. arXiv:2003.00628.
15. R. Martin-Martin et al., "Variable impedance control in end-effector space," IROS 2019. arXiv:1906.08880.
16. † M. H. Raibert and J. J. Craig, "Hybrid position/force control of manipulators," ASME Journal of Dynamic Systems, Measurement and Control, 1981. Volume and pages to be verified.
17. † M. T. Mason, "Compliance and force control for computer controlled manipulators," IEEE Transactions on Systems, Man, and Cybernetics, 1981. Volume and pages to be verified.
18. † N. Hogan, "Impedance control: an approach to manipulation," ASME Journal of Dynamic Systems, Measurement and Control, 1985. Volume and pages to be verified.
19. Y. Zhou et al., "On the continuity of rotation representations in neural networks," CVPR 2019.
20. X. Zhang et al., "Efficient sim-to-real transfer with online admittance residual learning," CoRL 2023. arXiv:2310.10509.
21. "Safe learning for contact-rich robot tasks: a survey," arXiv:2512.11908, 2025.
22. † J. Tobin et al., "Domain randomization for transferring deep neural networks from simulation to the real world," IROS 2017. To be verified.
23. † Y. Nakamura and H. Hanafusa, "Inverse kinematic solutions with singularity robustness for robot manipulator control," ASME Journal of Dynamic Systems, Measurement and Control, 1986. To be verified.
24. † C. W. Wampler, "Manipulator inverse kinematic solutions based on vector formulations and damped least-squares methods," IEEE Transactions on Systems, Man, and Cybernetics, 1986. To be verified.
25. † S. Chiaverini et al., "Review of the damped least-squares inverse kinematics with experiments on an industrial robot manipulator," IEEE Transactions on Control Systems Technology, 1994. To be verified.
26. † A. A. Maciejewski and C. A. Klein, "Numerical filtering for the operation of robotic manipulators through kinematically singular configurations," Journal of Robotic Systems, 1988. To be verified.
27. † T. Haarnoja et al., "Soft actor-critic: off-policy maximum entropy deep reinforcement learning with a stochastic actor," ICML 2018. To be verified.

*One further wording item needs the same treatment: the arms addressed here are described as "commanded through position and velocity interfaces" — chosen so as not to assert they cannot accept joint-torque commands. The Kortex low-level interface exposes a torque field whose availability and constraints must be confirmed against the vendor API reference before any stronger wording is used.*
