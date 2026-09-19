# Motivation

## The problem in plain language

A robot arm that can be told *where* to move is easy to buy. A robot arm that can be told *how hard to push* while it moves is not. Most collaborative robots on factory floors — including the Kinova Gen3 used in this project — expose position and velocity interfaces to the user. Contact-rich work (inserting a connector, seating a part, wiping a surface, deburring an edge) is exactly the class of task where "go here" is the wrong instruction: during insertion the thing that needs regulating is not the peg's position but the force it is applying and how the contact wrench is evolving. The distance you want to command is often smaller than your pose estimate's error.

So contact-rich automation is where factories still struggle, and reinforcement learning — while promising — is not yet industrially viable, mainly because of implementation cost and safety.

## Two barriers that compound each other

**Barrier 1 — the control interface.** Most recent force-learning results are demonstrated on torque-controlled research arms (Franka, Flexiv, and similar). The deployed fleet is dominated by cobots commanded through position and velocity. The point is *not* that these arms are physically incapable of joint-torque control — several expose a low-level torque channel — it is that the interface actually used in deployment, and the interface this work targets, is position/velocity. Bridging that gap means writing a compliance layer yourself.

**Barrier 2 — embodiment coupling.** When a policy maps sensor input straight to end-effector motion, the dynamics of that specific arm (mass, joint friction, servo bandwidth, kinematic chain) are absorbed into the network weights. Moving the skill to a new arm then requires:

- new contact data collected *on that arm*,
- new gradient steps, and
- a new checkpoint — which is a **new artifact**. Any safety evidence gathered for the old checkpoint no longer applies to it. If you certified a policy on arm A and fine-tuned it for arm B, you have not ported a certified policy; you have created an uncertified one.

## Why this matters now

- **Contact is where the fraction of the task that matters is growing.** As industrial targets move from pick-and-place toward assembly, the share of each task spent in contact goes up, and a position-only action space degrades in proportion.
- **Vision-language-action (VLA) models are strong at the semantic layer and weak at the contact layer.** A VLA can identify *which* socket and estimate an approximate pose; its pose-delta action space cannot express "press with eight newtons." A portable force-control skill is the layer such a planner would call for the last two centimetres. *(Statements about VLA action spaces are the author's reading of the field and should be checked against current sources before being repeated.)*
- **Cross-embodiment transfer in the literature is mostly attacked with data** — train on many robots and hope the model absorbs the differences. That scales with compute. An architectural alternative — keep the robot out of the weights entirely — scales with engineering hours instead. The two are complementary, not competing: a hand-written adapter that carries a portable safety argument is worth having even under a learned policy, because certification does not transfer through fine-tuning.

## The approach, in one paragraph

Put the robot **below an explicit interface** rather than inside the weights. A fixed observation vector goes up; six numbers come down — one desired force on the contact normal, five motion channels. Nothing robot-specific crosses the line: no joint angles, no torques, no motor currents, no manipulability. Below the line sits a robot-specific adapter: a 1 kHz analytic admittance reflex, a safety filter that counts every intervention, and a from-scratch damped-least-squares inverse kinematics. Porting to a new arm means rewriting only the adapter. The policy file is byte-identical, and its hash is published so anyone can check.

## What is *not* being claimed

- Force conditioning is not new.
- Timescale separation (slow policy, fast corrector) is not new — it is a settled result.
- Learning force control on a position-controlled arm is not new.
- **Hardware agnosticism is not claimed at all.** The claim is *architectural portability*, with real validation on one arm and cross-body evidence primarily in simulation.

The open part — and the contribution — is **freezing** the policy and measuring what survives a change of body across a *written* contract.
