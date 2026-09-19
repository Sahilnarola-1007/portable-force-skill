# Progress and Remaining Work

Status as of **19 September 2026**. All measured figures are from the real Kinova Gen3 or the lab PC unless marked otherwise.

Where the project stands in one line: **the engineering stack underneath the policy is built and measured; the policy itself trains next; the research result (the portability figure) comes after that.**

## 1. Summary by stage

| Stage | What it delivers | Status |
|---|---|---|
| A. Problem definition and design | Goal proposal, objectives, locked interface contract, literature review, system block diagrams | ✅ Done |
| B. Robot-specific adapter (below the contract) | 1 kHz loop, forward / inverse kinematics, admittance reflex, safety filter, mock-buildable stack | 🟡 Components done and measured in isolation; hardware integration in progress |
| C. Contract-side infrastructure | Phase machine, observation publisher, manifest, conformance test, log schema | ⬜ Not started (gated on B) |
| D. Simulation and training | MuJoCo environment, domain randomisation, SAC training, ONNX export | ⬜ Not started (gated on C) |
| E. Hardware validation | Frozen policy on the real Gen3, ~100-trial anchor campaign, baselines, policy-rate sweep | ⬜ Not started |
| F. Portability study | Ported adapters on simulated bodies, untuned-adapter series, ablations, primary figure | ⬜ Not started |
| G. Write-up and release | Paper, figures, public code release | ⬜ Not started |

## 2. Completed work

| Item | Result | Evidence |
|---|---|---|
| Six-DOF velocity-based admittance controller (~13 Hz) | Constant-force surface wiping holds ≈ 5 N with a steady-state band of ±0.5 N while tracing a lateral pattern at 0.02 m/s | Working on hardware |
| Diagnosis of the high-level loop limit | ~13 Hz; the Kortex twist command takes ~73 ms per round trip while the control mathematics takes 0.2 ms — the reason for the 1 kHz migration | Measured |
| Kinematics library | Classical-DH forward kinematics, numerical Jacobian, iterative damped-least-squares pose solver — written from scratch, FK residual vs. vendor pose 8–13 mm / ~1.5° (a known 7 mm tool-offset bookkeeping difference accounts for most of it) | Verified on hardware; unit-tested |
| 1 kHz low-level loop | Feedback read, position hold, single-joint sinusoid tracking | Working on hardware |
| Command integration scheme | Anchoring the command to the measured position leaked 96.5 % of a 5° command; anchoring to the previous command reproduced 5.0029° with 0.08° RMS error | Diagnosed and fixed on hardware |
| Real-time scheduling | SCHED_FIFO + CPU core pinning takes cycle-time standard deviation from 284 µs to 2.3 µs with zero deadline misses | Measured |
| Servo model (joint 7) | First-order fit, time constant ≈ 16 ms, ~10 Hz bandwidth — order of magnitude only | Inferred from two experiments |
| Low-level servoing mode | Joint *position* commands chosen over joint *velocity*: in velocity mode a joint coasts if command packets stop, with no fault raised; position mode stops because the target stops advancing | Measured on hardware, 17 Sep 2026 |
| Force sign convention | Pressing into the surface produces negative Fz; the controller consumes the negated value | Verified on hardware |
| Differential IK for the 1 kHz loop | Finite-difference Jacobian, LDLᵀ solve; mean 1.88 µs, p99.9 2.98 µs against a 1000 µs budget; no heap allocation on the hot path | Benchmarked, N = 10⁶ |
| Safety filter | Three clipping sites (force, Cartesian, joint) with an intervention counter; worst p99.9 ≤ 40 ns across four scenarios; counter invariants verified | 26 unit tests + benchmark |
| Mock Kortex SDK | Entire stack builds and tests with no robot attached (`-DUSE_KORTEX_MOCK`, default ON) | Builds |
| Perception pipeline ⚠ | YOLOv8 detection + FoundationPose (mesh-free) set up on the lab GPU; accuracy on the socket not yet measured | Set up; accuracy pending |

## 3. In progress

| Item | Purpose |
|---|---|
| Integrating reflex + safety filter + differential IK into the 1 kHz loop on hardware ⚠ | The integrated adapter; produces the cycle-time histogram under full load |
| Connector / socket fixture selection and procurement | Fixes the task, clearance, seating depth and time limit before the first reported run |
| Hardware verification items: raw F/T sign before software rotation, sensor noise floor, servo lag on the limited joints | Gate the contact phase machine thresholds and the safety bounds |

## 4. Remaining work

| Order | Item | Delivers |
|---|---|---|
| 1 | Contact test on hardware with the 1 kHz reflex | The O2 force band, measured against the ±0.5 N reference |
| 2 | Phase table and target source; task-frame latch | The phase one-hot and a stable task frame during contact |
| 3 | Manifest mechanism and observation publisher | The contract implemented end to end on hardware |
| 4 | Full-stack integration with a scripted dummy action; frozen log schema | Proof the loop closes; the log format used for every later trial |
| 5 | Public adapter repository, figures, force-hold video | The engineering artifact, released |
| 6 | MuJoCo environment implementing the contract; conformance test | Byte-identical observation/action encoding in sim and on hardware |
| 7 | Train version 1 (fixed force), 3 seeds; ONNX export + manifest + hash | The frozen policy |
| 8 | Deploy on the real Gen3; ~100-trial anchor campaign; baselines; policy-rate sweep | First real results; O1 anchor; O4 curve |
| 9 | Version 2 (policy-commanded force) | The full force-conditioned form |
| 10 | Ported adapters on simulated bodies; untuned-adapter series; ablations; second real body if lab access allows | The primary figure; test of the vz hypothesis |
| 11 | Write-up, figures, code release, submission | Paper (target: IEEE RA-L or IEEE CASE) |

## 5. Timeline

| Window | Focus |
|---|---|
| Late September 2026 | Integrated 1 kHz stack on hardware; contact test; contract-side infrastructure |
| October 2026 | MuJoCo environment; train and freeze version 1; deploy on the real Gen3; anchor campaign; baselines |
| Early November 2026 | Portability study across simulated bodies; ablations; version 2 |
| **Mid-November 2026** | **Write-up, figures and code release — expected project completion** |

The project — hardware campaign, portability study, write-up and code release — is expected to be complete by **mid-November 2026**. Under schedule pressure, objectives O1 and O2 are protected: the number of ported bodies is reduced before any baseline or ablation is cut, and hardware conditions are reduced before the number of trials within a condition, since a condition measured with too few trials cannot settle the claim.

## 6. How the existing code repositories fit

All code lives in public repositories at [github.com/Sahilnarola-1007](https://github.com/Sahilnarola-1007). This repository contains only documentation.

| Repository | Layer | Role |
|---|---|---|
| `kinova-kinematics` | Adapter | Classical-DH forward kinematics, Jacobian, damped-least-squares IK |
| `admittance-controller` | Adapter | Velocity-based admittance / force controller (the ~13 Hz wipe controller; reflex ported to 1 kHz) |
| `mae-sensor-driver` | Adapter / hardware | Driver and conditioning for the MAE 6-axis F/T sensor |
| `kinova-wrapper` | Adapter / hardware | Kortex low-level interface, 1 kHz cyclic loop, mock SDK |
| `surface-wipe`, `wipe-msgs` | Portfolio demo | Constant-force surface wiping (not a research task) |
| `kinova-moveit-bridge` | Tooling | MoveIt bridge used during earlier integration |
| `kinova-adapter` (planned) | Adapter | The integrated 1 kHz stack: safety filter → reflex → DLS IK, released as one buildable package |
