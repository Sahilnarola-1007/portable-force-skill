# Methodology

The system has three layers. Every component belongs to exactly one layer, and the layer decides what it is allowed to know.

| Layer | Runs at | Knows about the robot? | Contents |
|---|---|---|---|
| **Above the contract** | ~7–10 Hz | No | Perception (YOLOv8 + FoundationPose), task-frame builder, phase state machine (TargetSource) |
| **The contract — frozen policy** | ~30 Hz | **No — by construction** | SAC policy trained in MuJoCo, exported to ONNX, never modified after training |
| **Below the contract — adapter** | 1 kHz | Yes — everything robot-specific lives here | Safety filter, admittance reflex, damped-least-squares IK, forward kinematics, F/T sensor conditioning, Kinova low-level interface |

Colour key used in the diagrams: 🟥 hardware / sensors · 🟦 adapter (robot-specific, below the contract) · 🟪 policy side (portable bytes) · ⬜ infrastructure / offline.

---

## 1. System overview — sensors to joint command

Three sensors run at three rates: Gen3 joint feedback at 1 kHz (Kortex BaseCyclic), the MAE 6-axis F/T sensor at 500 Hz (contact reads as **negative Fz** in the tool frame), and RGB-D perception at roughly 7 Hz (~150 ms end-to-end target).

```mermaid
flowchart TB
    classDef hw fill:#f8d7d3,stroke:#c0392b,color:#000
    classDef adapter fill:#cfe8e6,stroke:#1a7f78,color:#000
    classDef policy fill:#e4d6f0,stroke:#7d3c98,color:#000
    classDef infra fill:#e8e8e8,stroke:#666,color:#000

    subgraph HW[Hardware]
        G3[Kinova Gen3 7-DOF<br/>BaseCyclic 1 kHz: q, q̇]:::hw
        FT[MAE 6-axis F/T<br/>500 Hz, tool frame, −Fz on contact]:::hw
        CAM[RGB-D camera<br/>YOLOv8 + FoundationPose]:::hw
    end

    subgraph AD[Adapter — robot-specific]
        FK[Classical-DH forward kinematics<br/>EE pose / twist in task frame]:::adapter
        WC[Wrench conditioning<br/>bias-corrected, kept in tool frame]:::adapter
        TS[TargetSource node<br/>phase + latched task frame]:::adapter
        OP[ObservationPublisher, 30 Hz<br/>38-element contract vector, manifest-normalised]:::adapter
    end

    CONTRACT{{Locked interface contract<br/>nothing robot-specific crosses this line}}:::infra

    POL[Frozen SAC policy, ONNX Runtime, ~30 Hz<br/>action = F*n, vx, vy, ωx, ωy, ωz<br/>never outputs vz]:::policy

    subgraph FAST[Fast path — 1 kHz]
        SF[SafetyFilter<br/>3 clipping sites, every intervention counted]:::adapter
        RX[Admittance reflex<br/>PI on −Fz → vz]:::adapter
        IK[DLS inverse kinematics<br/>finite-difference J, LDLᵀ solve]:::adapter
        CMD[Joint position command, 1 kHz<br/>q_send = q_send_prev + q̇·dt]:::adapter
    end

    G3 --> FK
    FT --> WC
    CAM --> TS
    FK --> OP
    WC --> OP
    TS --> OP
    OP --> CONTRACT --> POL
    POL --> SF --> RX --> IK --> CMD --> G3
    WC --> RX
    G3 --> IK
```

Two loops close: joint state and wrench return to the adapter at 1 kHz; the observation returns to the policy at 30 Hz. **Compliance comes from the reflex, not from the arm** — the Gen3 is driven through its position interface in this project.

---

## 2. The locked interface contract — observation up, action down

```mermaid
flowchart LR
    classDef tool fill:#fde2c8,stroke:#d35400,color:#000
    classDef task fill:#d6eaf8,stroke:#2874a6,color:#000
    classDef meta fill:#e8e8e8,stroke:#666,color:#000
    classDef policy fill:#e4d6f0,stroke:#7d3c98,color:#000
    classDef none fill:#fff,stroke:#999,stroke-dasharray:4 4,color:#555

    subgraph OBS[OBSERVATION — measured, 38 elements v1 / 40 v2]
        W[Wrench, last 3 samples · 18<br/>tool frame, N, N·m<br/>fixed sampling interval from manifest]:::tool
        P[EE position · 3<br/>task frame, m]:::task
        R[EE orientation, 6D · 6<br/>first two columns of measured rotation matrix]:::task
        V[EE velocity · 6<br/>task frame, m/s, rad/s]:::task
        PH[Contact phase one-hot · 5]:::meta
        V2[v2 only: force error, current F*n · 2<br/>tool z, N]:::tool
    end

    POL[Frozen policy<br/>SAC → ONNX, ~30 Hz]:::policy

    subgraph ACT[ACTION — policy output, 6 elements]
        FN[F*n · 1<br/>desired normal force, tool z, N]:::tool
        VT[vx, vy · 2<br/>tangential slide, task frame, m/s]:::task
        OM[ωx, ωy, ωz · 3<br/>angular velocity, task frame, rad/s]:::task
        VZ[vz · 0<br/>no channel — the reflex produces it]:::none
    end

    W --> POL
    P --> POL
    R --> POL
    V --> POL
    PH --> POL
    V2 -.-> POL
    POL --> FN
    POL --> VT
    POL --> OM
```

**Hard axis partition.** The contact-normal axis is force-controlled; every other axis is motion-controlled. Commanding force *and* velocity on the same axis is not merely discouraged — it is unrepresentable, because vz has no channel.

**Excluded on purpose:** joint angles, joint velocities, joint torques, motor currents, manipulability. All are robot-specific and live below the contract. Their absence is what lets the same policy bytes run on a different kinematic chain.

**Orientation** is observed as the first two columns of the measured rotation matrix. Because the action's orientation channels are angular velocities, no rotation is ever reconstructed from it and no orthogonalisation step exists anywhere in the loop.

**Versions.** v1 (38 elements) fixes the desired force to a manifest constant — the adapter overrides F*n. v2 (40 elements) lets the policy command F*n. The action is 6 elements in both.

---

## 3. The 1 kHz adapter loop — one policy action, from policy to arm

```mermaid
flowchart TB
    classDef hw fill:#f8d7d3,stroke:#c0392b,color:#000
    classDef adapter fill:#cfe8e6,stroke:#1a7f78,color:#000
    classDef policy fill:#e4d6f0,stroke:#7d3c98,color:#000

    A[Latest policy action, ~30 Hz<br/>read every 1 ms cycle]:::policy
    S1[Site 1 · clipForce<br/>scale F*n, sign preserved]:::adapter
    FT[MAE F/T, 500 Hz<br/>−Fz contact, tool frame]:::hw
    RX[Admittance reflex<br/>PI on −Fz → vz, measured dt, anti-windup]:::adapter
    TW[Assemble task twist<br/>vx, vy, vz_reflex, ωx, ωy, ωz]:::adapter
    S2[Site 2 · clipCartesian<br/>uniform scaling of tangential v and ω separately;<br/>vz excluded; workspace box refuses outward motion only]:::adapter
    IK[Rotate task → base, DLS IK<br/>finite-difference J, LDLᵀ solve of JJᵀ+λ²I, λ = 0.05]:::adapter
    Q[Measured q, q̇<br/>BaseCyclic feedback]:::hw
    MM[Manipulability monitor<br/>√det JJᵀ, NaN sentinel]:::adapter
    S3[Site 3 · clipJoint<br/>uniform q̇ scaling, joint-position refusal]:::adapter
    INT[Commanded-anchor integration<br/>q_send += q̇·dt; commanded-vs-measured gap guard]:::adapter
    OUT[BaseCyclic Refresh, 1 kHz<br/>SCHED_FIFO 80, pinned core, no heap allocation]:::hw
    IC[InterventionCounter<br/>all three sites log; target = 0]:::adapter

    A --> S1 --> RX --> TW --> S2 --> IK --> S3 --> INT --> OUT
    FT --> RX
    Q --> IK
    IK --> MM
    S1 -.-> IC
    S2 -.-> IC
    S3 -.-> IC
```

**Why the design looks like this**

- **Uniform scaling, not per-joint clipping** at Site 3: clipping joints individually would rotate the Cartesian direction of motion. Scaling the whole vector preserves it.
- **vz excluded from Site 2** so that the force-controlled axis has exactly one limiter (the reflex's own clamp), not two in series.
- **Commanded-anchor integration** (`q_send = q_send_prev + q̇·dt`): anchoring the command to the *measured* position instead leaked 96.5 % of the commanded amplitude on hardware. This was diagnosed as an interface-semantics error, not a gain problem.
- **Manipulability monitor**: below threshold → abort-and-retract in contact, joint-space fallback in free space. The all-zero joint configuration is a rank-3 singularity and is never used as a start pose.

**Measured on hardware / lab PC (as of September 2026)**

| Quantity | Value |
|---|---|
| DLS IK execution time (N = 10⁶, RelWithDebInfo) | mean 1.88 µs · p99.9 2.98 µs · budget 1000 µs |
| Safety filter execution time (four scenarios) | worst p99.9 ≤ 40 ns |
| 1 kHz cycle-time jitter, SCHED_FIFO + core pinning | stddev 284 µs → 2.3 µs, zero deadline misses |
| Servo model, joint 7 only (first-order fit) | gain ≈ 64 /s, time constant ≈ 16 ms (order of magnitude only) |
| Existing 13 Hz force controller (surface wiping) | holds ≈ 5 N within ±0.5 N |

---

## 4. Offline training and the portability claim

```mermaid
flowchart LR
    classDef infra fill:#e8e8e8,stroke:#666,color:#000
    classDef policy fill:#e4d6f0,stroke:#7d3c98,color:#000
    classDef adapter fill:#cfe8e6,stroke:#1a7f78,color:#000
    classDef hw fill:#f8d7d3,stroke:#c0392b,color:#000

    ENV[MuJoCo 3.8.1 environment<br/>Gen3 model, socket fixture, wrist force-sensor site<br/>first check: pressing into table gives −Fz]:::infra
    DR[Domain randomisation<br/>A: sensor noise from measured floor<br/>B: reflex mass/damping, servo lag, actuation delay, F/T offset & drift]:::infra
    SAC[SAC, Stable Baselines3, 3 seeds<br/>contract obs / action;<br/>reflex + filter sub-stepped inside the env]:::policy
    ONNX[ONNX export + manifest.yaml<br/>OBS_DIM, channel order, normalisation constants,<br/>wrench interval, force axis, F*n range, success criterion,<br/>safety bounds, contract version, SHA-256 of the ONNX]:::policy
    CONF[Conformance test + frozen log schema<br/>byte-identical obs/action encoding sim ↔ hardware]:::infra
    K7[Kinova Gen3 7-DOF adapter<br/>real hardware, ~100 classified trials]:::hw
    UT[Untuned-adapter series<br/>tests the vz hypothesis]:::adapter
    ARM2[Second-arm adapter<br/>simulation only]:::adapter

    ENV --> SAC
    DR --> SAC
    SAC --> ONNX
    ONNX --> CONF
    CONF --> K7
    CONF --> UT
    CONF --> ARM2
```

**Group B randomisation is what carries the portability claim.** Randomising only the environment (stiffness, friction, socket pose) would produce a policy robust to the task and fragile to the arm — the opposite of what is wanted.

**Same code path in simulation.** The reflex and the safety filter run *inside* the environment, sub-stepped at their real rate, not once per policy step. A conformance test checks that observation and action encodings are byte-identical between simulation and hardware before the policy is allowed to run.

**The working hypothesis.** Descent rate along the contact normal is the action channel most tightly coupled to arm mass, joint friction and servo bandwidth. Removing it from the action space (the reflex owns vz) is what is expected to let the skill cross bodies. This is a hypothesis; the untuned-adapter series is the measurement that tests it.

**Porting procedure (five steps per new body):** implement FK and differential IK for the new chain → define the task frame → re-tune admittance mass and damping → calibrate the F/T mounting and confirm the sign convention → set the safety bounds. Normalisation constants are explicitly *not* re-tuned; they are copied from the manifest unchanged. Person-hours per port are logged as a reported metric.

---

## 5. Perception, phase machine and the task-frame latch

```mermaid
flowchart TB
    classDef hw fill:#f8d7d3,stroke:#c0392b,color:#000
    classDef adapter fill:#cfe8e6,stroke:#1a7f78,color:#000
    classDef infra fill:#e8e8e8,stroke:#666,color:#000

    CAM[RGB-D frames<br/>camera on RTX 5070 host]:::hw
    YOLO[YOLOv8 detect<br/>socket region of interest]:::infra
    FP[FoundationPose<br/>mesh-free, zero-shot, ~150 ms target]:::infra
    TFB[Task-frame builder<br/>anchored at surface + goal, never at the robot base]:::adapter
    WD[Wrench + insertion depth<br/>debounced thresholds from measured noise floor]:::adapter
    PM[TargetSource phase machine<br/>5 phases + a recovery phase, debounced, refusal counter]:::adapter
    LATCH[Task-frame latch<br/>copied once on entry to CONTACT, held for the whole contact phase]:::adapter
    TF[/contract/task_frame<br/>→ FK, twist, observation/]:::infra
    PHO[/contract/phase<br/>→ one-hot into observation/]:::infra

    CAM --> YOLO --> FP --> TFB --> LATCH --> TF
    WD --> PM --> LATCH
    PM --> PHO
```

**Why the latch exists.** Perception jitters at roughly 7–10 Hz. Without the latch that jitter shows up in the observation as oscillation in the task-frame pose and velocity channels and — more damagingly — rotates the commanded tangential slide direction, so the search pattern chases the *frame* instead of the *socket*.

**Perception sits above the contract** and its only output is the task frame. Any 6-DOF pose source of comparable accuracy can be substituted on a new arm without touching the policy. Measured perception accuracy sets the initial pose-error range used in training.

**Known limitation.** The latch forbids updating the frame mid-contact, so curved-surface tasks (polishing, medical scanning) are out of scope and named as follow-on work.

---

## 6. Experimental plan (summary)

- **Task:** multi-pin connector insertion at sub-millimetre clearance. Success = full seating depth (from a hand-guided reference insertion) within tolerance, held for 1 s, within a per-trial time limit, with no safety abort. The criterion is geometric so it scores identically in simulation and on hardware; electrical continuity is a secondary check. Part, clearance, tolerance and time limit are fixed in the manifest before the first reported run.
- **Primary figure:** insertion success rate (with Wilson interval) vs. robot body, real Gen3 leftmost. Three series — frozen policy with tuned adapter, frozen policy with untuned adapter, baseline retrained per body — the frozen series reported with and without adapter-side residual adaptation. Simulated and real bodies drawn with distinct markers.
- **Baselines:** position-only policy with passive compliance; hand-tuned PI force control (the existing wipe controller re-tuned, with re-tuning effort reported); same policy with safety filter removed; high-rate baseline (same architecture at reflex rate, no separate fast layer).
- **Ablations:** remove force from the observation; remove the reflex; remove the safety filter; remove residual adaptation; sweep policy rate.
- **Rigor:** ≥100 hardware trials per condition that carries the primary claim (≥200 in simulation) — set by the ±15-point margin, not by convention; exploratory variants use 20 and are reported as intervals. Failures classified (missed entry, jam, stagnation, safety abort, timeout), not just counted. ≥3 seeds per policy, spread reported.
