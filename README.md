# ICPilot

<p align="center"><b>An AFL++ Variant for LinUCB-Guided Queue Prioritization, Phase-Bandit Action Selection, and Wrapper-Level Semantic Instrumentation</b></p>

<p align="center">
  Fuzzing-core component of a larger system stack, built on top of AFL++
</p>

---

## Abstract

**ICPilot** is a research-oriented AFL++ variant designed to improve exploration quality in structurally complex targets by combining **learning-based queue prioritization**, **phase-bandit-guided action selection**, and **wrapper-level semantic instrumentation support**. Rather than replacing AFL++'s mature coverage-guided execution backbone, ICPilot extends it in a minimally intrusive manner: candidate seeds can be re-ranked through a **LinUCB-style decision layer**, fuzzing effort can be steered through a **bandit-style phase/action selector**, and semantic signals emitted by upstream components can be bridged into the fuzzing process through compiler-wrapper support for markers such as `__POLAR_INS(...)`.

The project is intended for settings where pure coverage growth is not always sufficient to expose execution paths of higher analytical value. In such cases, a seed may be important not because it yields immediate local gains, but because it carries stronger long-horizon potential for unlocking deeper states, rare behaviors, or risk-relevant logic. ICPilot is therefore designed to preserve AFL++'s practical stability while improving its ability to prioritize **low-frequency but high-value execution opportunities**.

**Scope note.** This repository is **not** the complete system by itself, and it is **not** the offline-analysis module. It implements the **fuzzing core** in a larger end-to-end workflow. The full logical pipeline depends on companion repositories that provide additional capabilities such as target understanding, seed preparation, orchestration, debugging, and reporting.

---

## Motivation

Coverage-guided fuzzing remains highly effective for broad state-space exploration, but protocol-heavy or semantically structured targets often expose three recurring bottlenecks:

1. **Heuristic queue ordering is limited.** Inputs with the highest long-term exploratory value may not be selected early enough if ranking relies mainly on short-term heuristics.
2. **Fixed fuzzing behavior is often suboptimal.** Different execution states may benefit from different fuzzing actions or energy-allocation styles rather than a single static strategy.
3. **Coverage alone is semantically coarse.** Important logic often lies behind domain-specific predicates or state transitions that are not fully reflected by raw edge coverage.

ICPilot addresses these issues by integrating:

- AFL++'s original mutation and execution pipeline,
- a **LinUCB-inspired queue prioritization mechanism**,
- a **phase-bandit-guided action selection layer** for phase/energy control,
- and a **wrapper-level bridge** for semantic instrumentation markers.

The result is a design that stays close to AFL++'s operational model while enabling richer guidance for specialized fuzzing workflows.

---

## Method at a Glance

> **Note on the equations below.** The formulas in this section are **conceptual summaries** intended to describe the method at a paper-homepage level. They are not meant to be a line-by-line reproduction of the full implementation, and the concrete repository logic may differ in feature construction, reward shaping, update rules, and scheduling integration details.

### 1. Queue Prioritization as a Contextual Decision Problem

At a high level, ICPilot views queue selection as a contextual ranking problem. For a candidate seed with feature vector $x$, a LinUCB-style score can be written as:

$$
\operatorname{UCB}(x) = x^\top \hat{\theta} + \alpha \sqrt{x^\top A^{-1}x}
$$

where:

- $x$ represents seed-related features,
- $\hat{\theta}$ is the estimated reward parameter,
- $A^{-1}$ reflects uncertainty accumulated from historical observations,
- and $\alpha$ controls the exploration-exploitation trade-off.

In practice, this means ICPilot does not treat all queued seeds as equally promising. Instead, it favors candidates that are estimated to have both **high expected utility** and **high uncertainty worth exploring**.

### 2. Phase-Bandit-Guided Action Selection

Beyond queue ordering, ICPilot also models **how fuzzing effort should be applied** as a bandit-style decision problem. Instead of assuming a single best operational mode, the scheduler can choose among a set of **phase actions / bandit arms**, each corresponding to a different fuzzing tendency.

Conceptually, the action at round $t$ can be written as:

$$
a_t = \arg\max_{a \in \mathcal{A}} \left( \hat{\mu}_a + c \sqrt{\frac{\ln t}{n_a}} \right)
$$

where:

- $\mathcal{A}$ is the set of available phase actions,
- $\hat{\mu}_a$ is the empirical utility of action $a$,
- $n_a$ is the number of times that action has been selected,
- and $c$ controls exploration over under-used actions.

In ICPilot, these actions are not abstract labels only; they correspond to **different fuzzing guidance styles**. In the intended design, the arm set covers behaviors such as:

- **fast / deterministic-oriented exploration,**
- **balanced exploration,**
- **deep-path-oriented exploration,**
- and **risk-amplified exploration.**

That is, the phase bandit is meant to guide the **action style** or **energy-allocation tendency** applied to a seed, rather than merely reordering the queue again. This makes the LinUCB layer and the phase-bandit layer complementary: the former helps decide **which seed is more promising**, while the latter helps decide **how the current fuzzing effort should be spent**.

### 3. Reward Signal as Exploration Utility

A compact abstraction of the reward used by a learning-based prioritizer is:

$$
r_t = \lambda_1 \Delta \mathrm{cov}_t + \lambda_2 \Delta \mathrm{path}_t + \lambda_3 \Delta \mathrm{risk}_t - \lambda_4 \mathrm{cost}_t
$$

where the reward may combine gains in:

- coverage or new edges,
- newly activated paths or states,
- auxiliary semantic/risk-related signals,
- and the execution cost required to obtain those gains.

This form captures the core intuition behind ICPilot: a useful seed should not be judged only by immediate local coverage, but by its broader contribution to meaningful exploration.

### 4. Risk-Aware Guidance as an Auxiliary Signal

When semantic instrumentation is available, queue utility can be viewed conceptually as:

$$
S(q) = S_{\text{base}}(q) + \beta \cdot R(q)
$$

where $S_{\text{base}}(q)$ is the baseline scheduling utility and $R(q)$ is an auxiliary risk-oriented signal derived from instrumentation or downstream analysis. This makes it possible to bias fuzzing effort toward paths that are not only reachable, but potentially more valuable for later analysis.

### 5. Wrapper-Level Instrumentation Bridge

Instead of requiring users to manually patch each target, ICPilot extends the compiler-wrapper path so that semantic markers such as:

```c
__POLAR_INS(token)
```

can be bridged into the fuzzing workflow through the build toolchain. This reduces integration friction and allows upstream system components to emit supported markers without tightly coupling target source code to project-specific headers.

---

## Main Contributions

### C1. LinUCB-Enhanced Queue Prioritization

ICPilot introduces a dedicated scheduling mode via:

```bash
-p linucb
```

This mode is designed as an **extension of AFL++ behavior**, not as a detached scheduler replacement. The normal testcase queue and mutation lifecycle remain recognizable, while a learning-based layer reorders candidate seeds according to observed utility.

### C2. Phase-Bandit-Guided Fuzzing Actions

In addition to queue prioritization, ICPilot introduces a **phase-bandit guidance layer** that selects among different fuzzing action styles / phase arms. This layer is intended to control how fuzzing effort is allocated across fast, balanced, deep, or risk-amplified behaviors, making the execution strategy more adaptive than a fixed one-policy approach.

### C3. Wrapper-Level Support for Semantic Markers

The compiler-wrapper path is extended so that the project can automatically bridge markers such as `__POLAR_INS(...)` into the fuzzing process. This enables cleaner integration with upstream components in the larger system stack.

### C4. Minimal-Intrusion Extension Strategy

ICPilot deliberately preserves AFL++'s backbone instead of building a disconnected experimental runtime. This lowers migration cost, keeps the code path easier to evaluate, and helps the project remain practical as a reusable fuzzing core.

---

## Position in the Larger System

ICPilot should be understood as the **fuzzing engine component** in a broader system architecture.

This repository provides:

- the modified AFL++ runtime,
- the LinUCB-guided queue prioritization path,
- the phase-bandit-guided action-selection path,
- compiler-wrapper support for semantic instrumentation,
- and the execution-side logic required for guided fuzzing.

However, the **complete end-to-end logic chain** depends on companion repositories and services that implement surrounding functionality such as:

- target understanding or protocol interpretation,
- structured seed preparation,
- orchestration and task control,
- debugging and crash triage,
- and result organization or reporting.

Accordingly, this repository is best viewed as a **core execution module** within a larger system, rather than a standalone all-in-one platform.

---

## Repository Layout

The exact file set may evolve, but the modified logic is centered around the following areas:

- `src/afl-fuzz-linucb.c`  
  LinUCB-related queue prioritization logic.

- `src/afl-fuzz.c` and related fuzzing sources  
  Integration points connecting the prioritization path, phase actions, and AFL++'s execution loop.

- `src/afl-cc.c`  
  Compiler-wrapper integration for automatic bridge injection.

- `include/afl-risk-bridge.h`  
  Bridge header supporting semantic markers such as `__POLAR_INS(...)`.

- `include/config.h` and related headers  
  Compile-time switches and project-level configuration hooks.

---

## Build and Installation

ICPilot follows AFL++'s general build style and is intended primarily for Unix-like environments.

### Recommended Environment

- Linux (recommended)
- Clang / LLVM toolchain
- `make`, `gcc` or `clang`
- Python 3 for auxiliary tooling where applicable

For the LLVM-based instrumentation path, make sure the corresponding LLVM development tools are available.

### Build Matrix / Modes

The build system follows AFL++'s general workflow, but different build targets serve different practical purposes. The table below summarizes the most relevant modes for ICPilot.

| Mode / Command | Scope | Typical Use Case | Installs to System Path by Itself | Notes |
|---|---|---|---|---|
| `make all` | Main AFL++ binaries | Build the primary runtime/tool binaries with minimal extra distribution packaging | No | A practical default when the goal is to build the main executables first |
| `make source-only` | Source-oriented instrumentation path | Use when the workflow mainly focuses on source-code targets and compile-time instrumentation | No | Useful when the source-based fuzzing toolchain is the main concern |
| `make distrib` | Full distribution build | Build the broader AFL++ distribution, including the main project components in a more complete form | No | Best choice when preparing a fuller local build before installation |
| `make -j"$(nproc)" all` | Parallel main build | Faster multi-core version of `make all` | No | Recommended on Linux when build speed matters |
| `make -j"$(nproc)" source-only` | Parallel source-oriented build | Faster multi-core version of `make source-only` | No | Same semantics as `source-only`, but parallelized |
| `make -j"$(nproc)" distrib` | Parallel full distribution build | Faster multi-core version of `make distrib` | No | Common on development servers or multi-core workstations |
| `make LLVM_CONFIG=llvm-config-18 distrib` | Version-pinned full build | Use when multiple LLVM versions are installed and the intended one must be selected explicitly | No | Replace `llvm-config-18` with the correct binary on the host |
| `make all STATIC=1` | Static build variant | Use on supported platforms when static binaries are preferred | No | Availability depends on toolchain and platform support |
| `sudo make install` | Installation step | Install already-built binaries, wrappers, and related files into system directories | Yes | This is the actual system-wide installation step, not a build step |
| `make clean` | Cleanup | Remove previous build artifacts before rebuilding | No | Recommended after switching LLVM versions or modifying wrapper/instrumentation logic |

### Recommended Build Paths

For most Linux users, the practical build/install patterns are:

#### 1. Full Build + System-Wide Installation

```bash
git clone https://github.com/InvincibleMadman/ICPilot.git
cd ICPilot
make -j"$(nproc)" distrib
sudo make install
```

#### 2. Main Binaries Only

```bash
make -j"$(nproc)" all
sudo make install
```

#### 3. Source-Oriented Workflow

```bash
make -j"$(nproc)" source-only
sudo make install
```

### Build with an Explicit LLVM Version

If your system provides multiple LLVM versions, you can point the build to a specific `llvm-config` binary:

```bash
make LLVM_CONFIG=llvm-config-18 distrib
sudo make install
```

Replace `llvm-config-18` with the correct binary on your system. The same pattern also works with other targets, for example:

```bash
make LLVM_CONFIG=llvm-config-18 source-only
```

### Optional Build Variants

AFL++-style builds commonly also support variants such as:

```bash
make all STATIC=1
```

for static binaries on supported platforms.

### Clean Previous Artifacts

```bash
make clean
```

### Interpretation

From a project-usage perspective, `make all`, `make source-only`, and `make distrib` should be understood as **build modes**, whereas `sudo make install` is the **deployment step into the Linux system path**. In other words, `make distrib` alone does not complete system-wide installation; it prepares the build artifacts that can then be installed.

This distinction is especially important for ICPilot, since wrapper-level integration and instrumentation-related changes are often easiest to validate after a clean rebuild followed by an explicit installation step.

### Practical Notes

- `make distrib` is a **build** target, not the final system installation step by itself.
- If your goal is to place ICPilot / AFL++ tools into Linux system directories for regular shell usage, the usual sequence is **build first, then `sudo make install`**.
- If you rebuild after changing LLVM, wrappers, or instrumentation-related logic, cleaning first is often the safest option:

```bash
make clean
make distrib
sudo make install
```

---

## Basic Usage

### Compile a Target with the Wrapper

```bash
./afl-cc -o target target.c
```

or, if the wrapper is installed in your environment:

```bash
afl-cc -o target target.c
```

In ICPilot, the wrapper path is extended so that bridge support for semantic markers is wired in automatically. In the intended workflow, this reduces or eliminates the need to manually modify target headers just to support project-specific instrumentation markers.

### Run the LinUCB-Enhanced Fuzzing Mode

```bash
./afl-fuzz -i in -o out -p linucb -- ./target @@
```

Example:

```bash
./afl-fuzz -i seeds -o findings -p linucb -- ./target @@
```

This enables the LinUCB-guided prioritization path while preserving AFL++'s standard fuzzing lifecycle. In the project design, this mode can also cooperate with the phase-bandit guidance layer to decide not only which seeds are worth prioritizing, but also which action style should dominate the current fuzzing effort.

---

## Why This Design

### Why keep the AFL++ backbone?

Because AFL++ already provides a robust execution model, mature mutation machinery, and a workflow familiar to practitioners. ICPilot is designed as a practical extension, not a clean-room replacement.

### Why combine LinUCB with a phase bandit?

Because the two layers answer different questions. LinUCB estimates **which queued seed is worth more attention**, while the phase bandit estimates **which fuzzing action style is currently most useful**. Together they let the system adapt both seed choice and effort-allocation behavior.

### Why place the bridge in the compiler wrapper?

Because wrapper-level integration is easier to reuse, easier to automate, and less invasive than manually patching every target source file.

---

## Project-Style Summary

> ICPilot is a research-style AFL++ variant that augments coverage-guided fuzzing with LinUCB-guided queue prioritization, phase-bandit-guided fuzzing actions, and wrapper-level semantic instrumentation support. It serves as the fuzzing-core component of a larger system, preserving AFL++'s stable execution backbone while improving guidance quality for structurally complex targets.
