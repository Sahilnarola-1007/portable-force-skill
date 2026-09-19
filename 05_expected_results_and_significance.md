# Expected Results, Significance and Implications

> No success rate has been measured yet. Everything in this file is stated as an expectation or a hypothesis, and the narrative for both outcomes is written down **before** the numbers exist — so that what a result means is decided in advance, not after seeing it.

## 1. Expected findings

| Objective | Expected finding (hypothesis) | What it would look like in the results |
|---|---|---|
| **O2 — reflex** | The 1 kHz admittance reflex holds a commanded normal force in steady contact within ±0.5 N on the real Gen3, with the policy at ~30 Hz and differential IK inside the loop. | A force-tracking plot in contact; a cycle-time histogram under full load with the tail (p99.9) reported, not just the mean. |
| **O3 — safety filter** | Zero bound violations across every reported run, with every intervention counted. Under nominal operation the filter should almost never activate, because bounds are set outside the trained action range. | Intervention count per run, target zero. A non-zero count is itself a finding: the bounds and the training distribution disagree. |
| **O1 — portability (anchor)** | The frozen policy achieves a stated insertion success rate on the real Kinova Gen3 with zero gradient steps on the policy. ~100 classified trials, Wilson interval. | The leftmost point of the primary figure, with its interval. This number is the reference for everything else. |
| **O1 — portability (ported bodies)** | On simulated bodies with only the adapter re-implemented, success rate stays within the declared margin (proposed 15 points) of the anchor. | The remaining points of the primary figure: frozen-tuned, frozen-untuned, and retrained-baseline series, each with intervals; the same ONNX hash reported under every body. |
| **vz hypothesis** | Removing the descent velocity from the action space — the channel most coupled to arm mass, friction and servo bandwidth — is what lets the skill cross bodies. | The untuned-adapter series: if the policy still ports with an *untuned* reflex, the hypothesis is supported; if it collapses, the collapse localises where the seam leaks. |
| **O4 — policy-rate sweep** | The success-vs-policy-rate curve stays flat down to roughly 10–15 Hz for the analytic fast layer, and is steeper for a learned fast baseline. | The rate-sensitivity curve. No shape is claimed in advance. |
| **Port cost** | Roughly two to three weeks of engineering per new arm. | Logged person-hours per port, turning an estimate into a measurement. |

## 2. Both outcomes are reportable

- **If the frozen series stays within the margin:** the portability objective is supported. The result is a frozen force-control skill that moved to a different body across a written contract with zero target data and zero gradient steps, and a hash anyone can check.
- **If the frozen series collapses on ported bodies:** the result is still worth reporting — arguably more informative — because the *size and the axis* of the collapse show which quantity leaked through the seam and where the policy was silently relying on Kinova dynamics. That is a diagnosis, not a failure to report.

## 3. Why the results matter

### To the field

- **Cross-embodiment transfer is usually attacked with data.** This project attacks it architecturally: keep the robot out of the weights entirely. Measured evidence on whether a written interface is sufficient for a contact skill to cross bodies is missing from the literature (see `03_literature_review.md`, §3), in either direction.
- **The safety argument travels with the file.** Because the policy is byte-identical on every body, safety evidence gathered once does not have to be regathered after a port. Fine-tuning breaks this property; a written contract preserves it.
- **The contract is checkable.** Element count, ranges, units and frames can be unit-tested; the ONNX hash can be compared across bodies without access to the weights or the training data; the contract can be implemented by a third party who has neither.
- **It is a clean test of the vz hypothesis** — whether descent rate along the contact normal is *the* channel that couples a policy to an arm. The untuned-adapter series answers this regardless of which way the main result goes.

### To practitioners and industry

- **Force control on position/velocity-commanded cobots is a daily commercial problem** (assembly, insertion, sanding, deburring). The deployed fleet does not acquire torque control through a firmware update; the adapter runs on the interface that is already shipped.
- **Porting cost becomes engineering time, not data collection.** No teleoperation rig, no data pipeline, no data-governance answer — and the adapter is code a reviewer can read.
- **Every intervention is counted**, so "zero bound violations" is a number in a log, not a sentence in a paper.

### To learning-based robot stacks (VLA / foundation-model systems)

A vision-language planner is strong at deciding *which* socket and roughly *where* it is, and weak at *how hard to press* — its action space typically cannot express contact intent, and internet-scale pretraining contains no force data. *(Author's reading; verify against current sources.)* A frozen, portable force skill behind a fixed contract is the natural callable primitive for the last two centimetres: the planner emits a task frame, a desired force and a success criterion, and the contract-conformant skill does the rest at a rate the planner cannot reach. Nothing in this project has to change for that integration — the target source already takes a task frame and a desired force from above.

## 4. Potential contributions

| Type | Contribution | Status |
|---|---|---|
| Practical | A 1 kHz robot-specific adapter (from-scratch FK / DLS IK, admittance reflex, safety filter with counted interventions) that builds and tests without the robot attached | Components measured in isolation; integration in progress |
| Practical | A written, versioned, unit-testable interface contract plus manifest (normalisation constants, safety bounds, ONNX hash) as a portable skill format | Specified; manifest mechanism in progress |
| Methodological | A porting procedure with logged person-hours, so "two to three weeks per arm" becomes a measurement | Defined; first port not yet performed |
| Empirical | Real-hardware insertion results on a Kinova Gen3 with confidence intervals, classified failures, and a policy-rate sensitivity curve | Not yet measured |
| Theoretical / scientific | A test of the hypothesis that the contact-normal descent channel is what couples a learned contact skill to a specific body | Hypothesis stated; untuned-adapter series planned |

## 5. Stated limitations (carried into every presentation of the results)

1. Real validation is on **one arm**; cross-body evidence is **primarily simulation**, and simulation portability is not the same as real portability.
2. Every target arm requires a mounted, calibrated **external six-axis F/T sensor**. Its cost and integration effort are not counted in the adapter estimate.
3. The **contact gap** between simulation and hardware is the classic failure point for this class of work. Domain randomisation, bounded residual adaptation and hardware tuning reduce it; they do not remove it.
4. The safety filter is **engineered clipping**, not a formal certificate. A formal barrier is named as future work.
5. The **task-frame latch** rules out curved-surface tasks where the contact normal moves during contact.
6. **One task family.** Task breadth is what the next project is for.

## 6. Where this goes after the project (direction, not plan)

- **Year 1 (this project):** one skill, one contract, one real arm, measured portability.
- **Year 2:** a library of contract-conformant skills (insert, wipe, screw, mate, deburr) sharing one observation and action specification — a force-native skill API a planner can compose without knowing anything about the robot.
- **Year 3:** the logged wrench trajectories from every trial on every body become the beginning of a force-annotated corpus — the data that is currently missing for force-native learned action spaces.
