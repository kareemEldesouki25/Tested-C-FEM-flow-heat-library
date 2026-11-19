# SciML Learning Plan (mentor edition)

Nov 15, 2026 · @kareem

## Summary

The targets, six projects (P1–P6), order and dates of the [original plan](https://claude.ai/artifact/15dveFWzz5KgYz7KxPGAM9) stay. What changes is who does the work: Kareem and Ahmed write every line of project code and every analysis. Claude acts as the hiring manager and mentor who sets the bar and checks you cleared it.

1. **Objective:** come out in Feb 2028 able to defend every repo in a 45-minute technical interview without notes. The repos are the evidence; the skill is the goal.
2. **Claude's job:** a written spec with acceptance criteria every Monday, reading pointers, PR and results reviews, interview-style questions, and a pass/fail call at each gate.
3. **New per project:** learning objectives, concepts each person must explain unaided, and self-check questions.
4. **AI rules:** AI may write glue, plots and boilerplate you already understand. It may not write core numerics, test design or validation judgment.
5. **Week 1** is a spec for each of you below, with no code in it.

## How we work

You build; Claude reviews like the person who would hire you. A deliverable counts only when its owner can explain it and it passes the spec.

| Who | Does | Does not |
| --- | --- | --- |
| Kareem | Writes the numerics, physics checks, validation studies and tech notes for his part | Hand his core numerics to AI or to Ahmed |
| Ahmed | Writes the software architecture, bindings, tests, CI, pipelines and services for his part | Hand his core design to AI or to Kareem |
| Claude | Writes specs and acceptance criteria, points to reading, reviews PRs and plots, asks interview questions, calls gates pass or fail, says when a goal is unrealistic | Write project code, templates, scaffolds, tests or analysis. It gives a snippet only when you ask for one specific thing, and labels it |

**Gates.** Each project has a gate review in its last week. Claude asks 5–8 questions per person from that project's concept list, with no notes, and reviews the repo against its spec. Pass = acceptance criteria met and every concept answered. Fail = you fix the gaps in the buffer week. A failed gate never becomes a post or a CV bullet.

**Cross-learning.** Each of you reviews the other's PR before Claude does, and must be able to explain one concept from the partner's list per project. This replaces the old "learns along the way".

## Rules for AI use

The test is simple: use AI only for work you could already write and explain yourself, and that is not the skill the project is meant to prove. This follows the plan's AI-era filter: spend AI on what is getting cheaper, spend your own hours on what is gaining value.

| Fine to use AI (and say so in the PR) | Not allowed (you write it) |
| --- | --- |
| Plotting, file parsing, log parsing, format conversion | FEM assembly, discretisation, solvers, boundary conditions, time stepping |
| Boilerplate you have written by hand once before (e.g. a second CMake target, a FastAPI route) | The first CMake layout, the pybind11 API design, the package structure |
| Explaining an error message, an unfamiliar API, or a concept, as a tutor | Test design: what to test, the tolerances, the reference values |
| Translating your own working code between languages after you validated it | Validation judgment: is the result physically right, why it fails, what the error bars mean |
| Docstrings and README wording, after the content is yours | Loss functions, model architecture choices, UQ method, gradient checks |
| The LLM inside P3's agent (that is the product) | P3's verification gates and their thresholds |

Three habits:

- **Label it.** Any AI-written lines go in a commit or PR note: what, which tool, why. Claude will ask about them in review.
- **Explain or delete.** If you can't explain an AI-written line in review, rewrite it yourself.
- **Try first.** Spend 30 minutes stuck before asking AI, and ask for a hint or a concept, not the fix.

## Weekly mentor loop

Every week runs the same five steps, so 8 hours each always go to building and the 2-hour joint session goes to review.

1. **Monday: spec.** Claude posts one spec per person: goal, acceptance criteria, reading (with chapter or page), and 2–3 questions you should be able to answer by Friday.
2. **Mon–Thu: build alone (about 6 h each).** Ask Claude for concepts and hints, not code. Log what you got stuck on.
3. **Thursday: open a PR or post results.** Include a short note: what you did, what you checked, what you are unsure of, any AI-written lines.
4. **Before the joint session: written review.** Your partner reviews first, then Claude reviews the PR and plots, and leaves questions.
5. **Joint session (2 h).** 30 min explain-back (each person explains the other's work), 60 min fixing review points together, 30 min agreeing next week's scope. Claude adjusts the next spec to what you found hard.

Skipped a week? Say so on Monday and Claude shrinks the spec rather than letting the plan drift.

## Corrections to the original plan

- **13 vs 17 job descriptions.** Your prompt says 13 JDs are attached; the skill-gap matrix counts "of 17 postings" because some entries hold more than one role. I have not recounted, so the counts in that matrix are unverified. Fix: list the postings in one table (company, role, link) and recount the top 6 skills yourselves. That is a good first exercise in reading JDs like a hiring manager.
- **"Learns along the way"** in the P1 split is replaced by a named concept each person must explain (see projects).
- **Week 1 "repo template"** is now Ahmed's own build task, not something Claude provides.
- **Ahmed's Physics AI application now** stays; it does not depend on the portfolio.

## Projects: what each of you must learn

Scope, benchmarks, stacks and fallbacks are unchanged from the original plan. Below is what each project must teach, and how the gate checks it.

### P1 — Tested C++ FEM flow + heat library (6 weeks)

- **Learning objectives:** turn a research code into a library with a stable API; design a regression suite; add and verify a new physics module.
- **Kareem explains unaided:** GLS stabilisation and why plain Galerkin fails at high Re; Boussinesq approximation and its limits; mesh-convergence order from three meshes; why Ghia is a regression test, not a proof.
- **Ahmed explains unaided:** CMake targets and transitive dependencies; pybind11 ownership and copies of Eigen arrays; what a weak-form assembly loop does; why the CI matrix covers Windows and Linux.
- **Self-check:** What tolerance did you pick for the Ghia test, and why not tighter? If someone changes the quadrature order, which test fails first? What does your API promise not to change in v0.2?

### P3 — Agentic OpenFOAM workflow with physics gates (5 weeks)

- **Learning objectives:** design checks that catch a confident wrong answer; structure an LLM tool-calling loop; build an evaluation harness.
- **Kareem explains unaided:** what checkMesh's non-orthogonality and skewness mean for the solution; when residuals lie; a mass/energy balance check and its threshold; y+ and wall functions.
- **Ahmed explains unaided:** the tool-calling loop and where state lives; how the harness injects faults; the false-positive vs false-negative trade-off of each gate.
- **Self-check:** Which injected error got past every gate, and why? What would make a gate threshold case-dependent? Would you trust the agent without the LLM explaining its edits?

### P2 — Airfoil surrogate service with uncertainty (8 weeks)

- **Learning objectives:** build a reproducible data-to-model pipeline; quantify and test uncertainty; detect out-of-envelope inputs.
- **Kareem explains unaided:** computing Cl/Cd from surface pressure and shear; why RANS data has its own error; what extrapolation means in Re and angle of attack; how to read a calibration plot.
- **Ahmed explains unaided:** why a deep ensemble gives a spread and what it misses; graph vs point-cloud input choices; DVC vs MLflow roles; ONNX export pitfalls.
- **Self-check:** Does your ensemble spread actually grow on the extrapolation split, by how much? What would a design engineer do with a flagged prediction? What is the error of the solver data itself?

### P4 — 0D/1D cooling loop, FMU, UDE correction (7 weeks)

- **Learning objectives:** acausal system modelling; FMI co-simulation; hybrid physics + ML components.
- **Kareem explains unaided:** acausal vs causal models; how a pump curve and pipe losses set the operating point; energy balance closure; a PI controller's response to a load step.
- **Ahmed explains unaided:** model exchange vs co-simulation FMUs; what a UDE learns and what stays physics; why training on P1 data is multi-fidelity and what can go wrong.
- **Self-check:** Where does your energy balance not close, and why? Does the UDE extrapolate outside the P1 Ra range? What would you hand another team: the FMU or the source?

### P5 — Test-data calibration on a differentiable model (7 weeks)

- **Learning objectives:** write a differentiable solver; verify gradients; solve an inverse problem with honest uncertainty.
- **Kareem explains unaided:** forward vs reverse-mode AD; a finite-difference gradient check and its step size; identifiability; why two parameters can trade off.
- **Ahmed explains unaided:** JAX jit, vmap and grad on your model; how confidence intervals are computed (e.g. Laplace or bootstrap); GPU profiling result and the bottleneck.
- **Self-check:** Which parameter is hardest to recover and why? How does your tool tell the user the data cannot pin a parameter? What noise level breaks it?

### P6 — Live thermal digital twin (12 weeks, joint)

- **Learning objectives:** integrate models with live data; online recalibration; fault detection; demo a system end to end.
- **Kareem explains unaided:** how the P4 model maps to the rig; prediction error budget at 10 minutes; what drift vs fault looks like in residuals.
- **Ahmed explains unaided:** the data path from thermocouple to dashboard and its latency; how replay mode works; what changes at production scale.
- **Self-check:** What does the twin do when a sensor drops out? How did you prove the fault flag is not just noise? Which part would you cut if you had half the time?

## Roadmap and gates

Dates and order are unchanged: 23 Nov 2026 to 27 Feb 2028, one project at a time, buffer weeks between projects. The week-by-week Phase 1 table and the Phase 2–3 sprints in the original plan still apply; each row becomes a Monday spec. Each phase gate now has a learning check on top of the original exit criterion.

| Gate | Date | Original exit criterion | Added learning check |
| --- | --- | --- | --- |
| 1 | 28 Feb 2027 | 2 repos public with green CI; 4 posts; applied to Siemens Physics AI, UnityX, AE London | Each of you passes a 45-min mock technical interview on P1 + P3 with Claude, no notes |
| 2 | 29 Aug 2027 | 5 repos and 5 tech notes public; about 10 posts; applied to Siemens ROM and Airbus | Mock interview on P2, P4, P5; each person whiteboards one partner-owned concept per project |
| 3 | 27 Feb 2028 | Twin demo video; one merged open-source PR; Tier 2 applications | Full system-design interview on P6 plus a live debugging task on one of your own repos |

A post or CV bullet goes out only after its project gate passes.

## Week 1 specs (23–29 Nov 2026)

Kareem's week is light because the SU2 application is due 27 Nov. Ahmed builds the repo template both of you will use from P1 on. Claude reviews both before the joint session.

### Kareem (8 h)

**Goal:** submit a strong SU2 application, and run your first acausal model.

- **Task A, SU2 application (6 h).** You write the draft. Share it by Wed 25 Nov; Claude reviews it as a hiring panel would and returns questions, not rewrites. Submit by Thu 26 Nov, a day before the deadline.
- **Task B, Modelica by Example ch. 1 (2 h).** In OpenModelica, write from scratch a first-order system and a Newton's-law-of-cooling model, then add a second thermal mass.
- **Acceptance criteria:**
  - Application submitted, with its specific SU2 contribution (what you changed in the source, why it mattered) stated in your own words.
  - Cooling model's steady state matches your hand calculation; one plot with both masses.
- **Reading:** [Modelica by Example, Basic Equations chapter](https://mbe.modelica.university/behavior/equations/).
- **Be ready to answer:** What does "acausal" mean in your two-mass model: which equations did you not have to rearrange? What time constant did you expect, and what did you get?

### Ahmed (8 h)

**Goal:** a repo template that builds a tiny C++ library, binds it to Python, and tests both in CI. You design and write all of it.

- **Task A, shared GitHub org (0.5 h).** Create it; both of you are owners; protect `main` so every change goes through a PR with one review.
- **Task B, template repo (7.5 h).** One trivial C++ function of your choice (e.g. a trapezoid-rule integrator), exposed through pybind11, built with CMake, tested with a C++ test framework and pytest, run in GitHub Actions on Ubuntu.
- **Acceptance criteria:**
  - A fresh clone builds and passes all tests with the commands in the README, and nothing else.
  - CI runs on every PR. One PR shows a deliberately broken test turning CI red, then the fix turning it green.
  - The integrator has a test against an exact answer, with a tolerance you justify in the PR.
  - The README states what is out of scope for now (Windows, wheels, coverage).
- **Reading:** [An Introduction to Modern CMake](https://cliutils.gitlab.io/modern-cmake/), [pybind11 First steps](https://pybind11.readthedocs.io/en/stable/basics.html), [GitHub Actions quickstart](https://docs.github.com/en/actions/writing-workflows/quickstart).
- **Be ready to answer:** Why `target_link_libraries` with PUBLIC vs PRIVATE? Where does the compiled Python module end up, and how does pytest find it? What would break first on Windows?

### Joint session (2 h)

- Kareem reviews Ahmed's template PR; Ahmed reviews Kareem's Modelica model. Each explains the other's work back.
- Recount the job descriptions together (one table: company, role, link) and settle the 13 vs 17 question.
- Agree a PR note format (what, checked, unsure, AI lines). Claude then posts the Week 2 specs.

## Decisions still open

1. **Ahmed on Julia:** commit to the side track for P4, or move the UDE to PyTorch. Decide by the end of P1.
2. **Budget:** P6 rig hardware, and an LLM API budget for P3 or a small local model. Decide before P3 starts (25 Jan 2027).
3. **Review depth:** should Claude review every PR, or only one PR per person per week? Every PR is better for learning but costs more of your joint time.

Note: the reading links in Week 1 could not be opened from here, so check each one resolves; if a page has moved, search the site for the chapter name.
