# Porting a Frozen Force Policy Across Cobots

**A locked interface contract for cross-embodiment insertion on position/velocity-commanded cobots**

Sahil Narola · Advanced Biomechatronics and Locomotion Laboratory, Carleton University · Supervisor: Prof. Mojtaba Ahmadi

This repository is the **documentation front door** for the project. It contains no code. Its purpose is to let a first-time reader understand the goal, the approach, what has been measured, and what remains — in under fifteen minutes.

> **In one sentence:** a frozen force-control skill moves to a new robot arm without a single gradient step, because the robot lives *below an explicit interface* rather than *inside the network weights*. The policy file is byte-identical across bodies, and its SHA-256 hash is published so anyone can check.

## Read in this order

| # | File | What it answers |
|---|---|---|
| 1 | [Title, abstract, goal and objectives](01_title_abstract_goal_objectives.md) | What is the claim, and what decides whether it holds? |
| 2 | [Motivation](02_motivation.md) | Why does this problem matter, and why now? |
| 3 | [Literature review](03_literature_review.md) | Where does this sit relative to existing work, and what gap does it fill? |
| 4 | [Methodology](04_methodology.md) | How is the system built? (block diagrams) |
| 5 | [Expected results and significance](05_expected_results_and_significance.md) | What is expected, why it matters, and what is *not* claimed |
| 6 | [Progress and remaining work](06_progress_and_remaining_work.md) | What is done, what is next, expected completion |

## Claim boundary

- **Architectural portability**, not hardware agnosticism.
- **Real validation on one arm** (Kinova Gen3 7-DOF); cross-body evidence primarily in simulation.
- Every target arm requires an **external six-axis force/torque sensor** — a stated precondition.
- The safety filter is **engineered clipping with counted interventions**, not a formal certificate.

## Code

All code is in separate public repositories at [github.com/Sahilnarola-1007](https://github.com/Sahilnarola-1007). The whole stack builds and tests **without the robot attached** (mock SDK, default ON). See [§6 of the progress file](06_progress_and_remaining_work.md#6-how-the-existing-code-repositories-fit) for how each repository maps to the architecture.

## Status

Engineering artifact (1 kHz adapter) built and measured; policy training and the portability study follow. Expected completion: mid-November 2026. Target venue: IEEE RA-L or IEEE CASE.
