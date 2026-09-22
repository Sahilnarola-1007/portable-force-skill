# Porting a Frozen Force Policy Across Cobots
## A Locked Interface Contract for Cross-Embodiment Insertion

**Sahil Narola** — Advanced Biomechatronics and Locomotion Laboratory (ABL), Carleton University, Ottawa, Canada
Supervisor: Prof. Mojtaba Ahmadi
Status: graduate research project, in progress (document date: 19 September 2026)

---

## Abstract

Contact-rich assembly tasks such as connector insertion are hard to automate because success depends on regulating contact force, not on following a position trajectory. Most recent force-learning research runs on torque-controlled research arms, while the deployed industrial fleet is dominated by cobots commanded through position and velocity interfaces. A second barrier is that a learned contact skill is normally tied to the arm it was trained on, because the robot's dynamics are absorbed into the network weights; moving it to a new arm costs new data, new gradient steps, and a new checkpoint whose earlier safety evidence no longer applies.

This project targets the second barrier. A force-conditioned policy is trained once in simulation and then frozen. It communicates with the robot only through a **locked interface contract** — 38 observation elements up (40 in version 2) and 6 action elements down — in which no robot-specific quantity appears. Everything that depends on the robot body (kinematics, a 1 kHz admittance reflex, force-sensor calibration, safety bounds) lives below that contract inside an engineered **adapter**. The goal is to show that the same frozen policy performs a contact-rich insertion on a different robot body after the adapter alone is re-implemented, with zero target-robot data and zero gradient steps on the policy. The method assumes an external six-axis force/torque sensor at the wrist of every target arm; this is a stated precondition, not a result. Validation is on a real Kinova Gen3 7-DOF. Cross-body evidence is planned primarily in simulation, with one additional real body of the same family (Gen3 6-DOF) as a stretch objective. The claim is **architectural portability, not hardware agnosticism**.

---

## Goal statement

> A locked interface contract lets one frozen force-conditioned policy perform a contact-rich insertion task across kinematically different cobots driven through position and velocity interfaces, with an analytic admittance reflex and a bounded safety filter below the contract. The policy is validated on a real Kinova Gen3 7-DOF. Cross-body transfer is evaluated primarily in simulation, with one additional real body of the same family as a stretch objective — in every case with zero target-robot data and zero gradient steps on the policy.

In one sentence: **the robot lives below an explicit interface, not inside the network weights — so porting the skill means rewriting the adapter, and the policy file is byte-identical (its SHA-256 hash is published for every body).**

---

## Objectives

The four objectives are not independent claims. O1 is the claim the project stands or falls on; O2 is the mechanism without which O1 cannot exist; O3 is the guarantee that makes the result safe to run and safe to report; O4 measures how much of O1 the mechanism is actually responsible for. They are listed in dependency order.

| # | Objective | Role | Success criterion (what decides it) | What would falsify it |
|---|---|---|---|---|
| **O1** | A frozen force skill ports to a new robot body | Primary claim | The same trained policy file, never retrained, is first scored on the real Kinova Gen3 (the reference), then on each other robot. It passes if success on the other robot is no more than **15 percentage points** below the Gen3 score (proposed margin — see "O1 in detail" below). | Success on the other robots drops by more than the margin and cannot be recovered by tuning the robot-specific adapter. This is still a useful result: which failure appears shows what robot-specific information the policy was secretly relying on. |
| **O2** | An analytic reflex carries the robot body below the contract | Enabling mechanism | On the real arm, the 1 kHz admittance reflex holds the commanded normal force in steady contact to a band no wider than ±0.5 N (the band already measured for the existing wipe controller), with the policy at ~30 Hz and differential IK inside the loop. Loop jitter under full load reported alongside. | Force regulation that is stable only when the policy runs fast — meaning the reflex is not absorbing contact transients and the timescale split is decorative. |
| **O3** | A bounded safety filter sits between the policy and the robot | Rigor requirement | Zero violations of the declared force, Cartesian-velocity, workspace and joint bounds across every reported run on every body. Every intervention is counted and logged. Worst-case filter execution time measured and shown to fit inside the 1 kHz budget. | Any limit violation in a reported run, or a worst-case execution time that does not fit the 1 kHz budget. |
| **O4** | Policy rate is swept and reported | Required ablation | Success rate reported as a function of policy rate, for the analytic fast layer and for at least one alternative fast layer under matched conditions. The curve is the deliverable; no shape is claimed in advance. | Success drops sharply when the policy is slowed down. That would mean the policy, not the 1 kHz reflex, is doing the fast force control — which weakens O2 and would be reported as such. |

**If the schedule slips, what gets cut first:** O1 and O2 are never dropped. The first thing reduced is the *number of robots* the policy is ported to (for example, three simulated arms become one). The comparison methods (baselines) and the remove-one-component tests (ablations) are kept, because without them a success rate cannot be interpreted: there is nothing to show it beats a simpler approach, or which component is responsible for it.

### O1 in detail — how to read it

**The test in three steps.**

1. **Reference score.** Train the policy once in simulation, freeze it, and run ~100 insertion trials on the real Kinova Gen3. Suppose it succeeds 80 times: the reference is 80 %.
2. **Port.** Move to a different robot. Rewrite *only* the adapter — the robot-specific code below the contract (kinematics, force reflex tuning, sensor calibration, safety limits). The policy file is not touched: no new training data from that robot, no retraining.
3. **Compare.** Run the same trials on the new robot. With a 15-point margin and an 80 % reference, the port passes if the new robot scores 65 % or higher.

**Why compare to a reference instead of stating a fixed target?** "Portable" is a relative claim — it means *the skill works about as well on robot B as on robot A*. Without a measured score on robot A, there is nothing to compare robot B against.

**Why 15 percentage points?** Two constraints set it:

- *It must be measurable with a realistic number of trials.* Each success rate has sampling noise. Assuming ~80 % success on both robots, the 95 % uncertainty on the *difference* between two rates is about ±25 points at 20 trials each, ±16 at 50, ±11 at 100, and ±8 at 200 (normal approximation). A 15-point margin can therefore be resolved with ~100 trials per robot on hardware and ~200 in simulation. A 5-point margin would need well over a thousand trials per robot — not feasible on a real arm.
- *It must still mean something.* A 30-point allowance would call almost any transfer a success.

15 is a judgement call between those two limits, not a derived constant. What matters is that it is **fixed with the supervisor before any trial is run**, so the pass/fail line cannot be moved after the results are seen.

**How the claim is kept honest.**

- **With and without adapter tuning.** The adapter may include a small online correction to the force reflex that adapts to the new robot while running. It never changes the policy. Every result is reported both with and without this correction, so a reader can see how much of the success comes from the frozen policy itself.
- **Proof the policy file is unchanged.** The policy is a single ONNX file. Its SHA-256 hash (a fingerprint that changes if even one byte changes) is published next to every robot's result. Identical hashes prove the same file ran everywhere.

---

## Scope boundaries (stated up front)

- Real validation is on **one arm** (Kinova Gen3 7-DOF). Cross-body evidence is **primarily simulation**.
- Every target arm must carry a mounted, calibrated **external six-axis F/T sensor**. Arms without one are out of scope.
- The safety filter is **engineered clipping with counted interventions**, not a formal certificate.
- **One task family** (connector insertion). Surface wiping is retained as a portfolio demonstration only.
