# Infralytix — Guide Briefing & Q&A Prep
### For tomorrow's project explanation / conference-application defense
---

## 1. The 60-Second Pitch (lead with this)

> "Infralytix implements the Advanced Cost-Aware Max–Min scheduling algorithm from a 2024 Cluster Computing paper — it schedules dependent cloud workflow tasks onto virtual machines to minimize both makespan and cost. The paper's own contribution was purely algorithmic, tested via a Python script. We built a full web platform around it: users upload a workflow as a DAG, configure a VM pool, and run the algorithm — plus three baseline algorithms for comparison — with a live α slider trading off speed against cost. Everything paper-derived is kept explicitly separate from what we engineered on top, so the algorithm's fidelity to the published research is independently auditable."

---

## 2. Full Walkthrough — Say It In This Order

### 2.1 The Problem
Cloud task scheduling assigns tasks to VMs to minimize makespan (total time) and cost. It's NP-hard — brute force is O(m^n) for n tasks, m VMs. Classical algorithms (FCFS, Round Robin, SJF, Max–Min, Min–Min) each optimize only part of the picture, and none handle **dependent** tasks with cost-awareness well.

### 2.2 The Research Foundation (cite this precisely)
Raeisi-Varzaneh, Dakkak, Fazea, Kaosar. *"Advanced cost-aware Max–Min workflow tasks allocation and scheduling in cloud computing systems."* Cluster Computing, 27, 13407–13419 (2024).

Say explicitly: *"This paper is our algorithmic foundation. Everything about task-selection and VM-selection logic comes from Algorithm 2 of this paper — we did not modify it."*

### 2.3 The Algorithm — explain it in plain language, then formulas

**Plain language:** "At each step, among tasks that are ready to run (no unfinished dependency), the algorithm picks the one whose own size *plus* its children's total size is largest — not just its own size. That one change is what lets independent parts of a workflow run concurrently instead of one-at-a-time, which was the flaw in the paper's own baseline (Workflow Max–Min). Then, depending on a cost-management flag, the task either goes to the fastest available VM, or to whichever VM minimizes a weighted mix of normalized time and normalized cost — that weight is your α, tunable from 0 (cheapest VM wins) to 1 (fastest VM wins)."

**Formulas (know these cold):**
- Makespan: `MS = max(MS(VM_1), …, MS(VM_k))`
- Resource Utilization: `U = (Σ MS(VM_i) / MS) × (1/k)`
- Idleness Cost: `IC = Σ (MS − MS(VM_i)) × Cost(VM_i)`
- VM selection (cost-aware): minimize `α × NormalizedCompletionTime + (1−α) × NormalizedCost`

**Reference numbers from the paper (cite if asked):** average waiting time improved from 5.27s to 4.67s; core utility improved up to 34%; idleness cost reduced up to 92% (122→10 units on one task set); complexity stays O(mn²), same order as classical Max–Min — the paper's contribution is schedule *quality*, not asymptotic speed. Worked example: 3 VMs at 5000/3000/2000 MIPS, makespan converges to 5.9s.

### 2.4 Tech Stack (with the "why" — guides always ask why)
- **React** — component-based UI for dashboards, sliders, live comparison charts
- **Node.js + Express** — lightweight REST layer, native JSON
- **Python + NetworkX** — matches the paper's own reference implementation language; NetworkX gives cycle detection and parent/child extraction for free
- **MySQL** — relational schema (users → workflows → tasks → dependencies → runs → assignments) is naturally foreign-key-heavy
- **JWT + bcrypt** — stateless, horizontally scalable auth; adaptive password hashing

### 2.5 Architecture (draw this if you can)
Layered: React client → Express API (JWT middleware, validation, controllers) → Scheduler Bridge (spawns/invokes the Python engine) → MySQL. Single logical backend service — deliberately not microservices, per project scope constraint.

### 2.6 What's Actually Novel (the "convince him" part — see §4, this needs work)

### 2.7 Current Status — be honest
SRS and SDD are complete and internally consistent (after tonight's fixes). The core algorithm is implemented and unit-tested in isolation, including cyclic-graph rejection and reproduction of the paper's own worked-example makespan. Full-stack integration (upload → schedule → dashboard) is in progress. Say this plainly if asked — a guide respects an honest "here's what's done, here's what's next" far more than an overstated "it's all working."

---

## 3. Anticipated Q&A

**On the algorithm:**
- *Q: Why does the task-selection rule include children's execution time?* — A: Because selecting purely by a task's own size (the older Workflow Max–Min) causes independent applications to run sequentially instead of concurrently — the paper demonstrates this with a two-application example. Including children's time captures downstream impact.
- *Q: What happens when α = 0 or α = 1?* — A: α=1 behaves identically to the non-cost-aware mode (fastest VM always wins). α=0 sends everything to the cheapest VM regardless of speed — the paper's own sensitivity table shows makespan spikes sharply as α approaches 0.
- *Q: What's the time complexity, and why isn't it better than classical Max–Min?* — A: O(mn²), same order — because the algorithm doesn't exhaustively search every task/VM combination at the fetch step, same shortcut classical Max–Min takes. The contribution is schedule quality (makespan, cost, utilization), not asymptotic speed.
- *Q: What happens with a cyclic dependency?* — A: Rejected at upload/parse time with the specific offending edge — never reaches the scheduler.

**On tech/architecture:**
- *Q: Why Python for the scheduler instead of just doing everything in Node?* — A: The paper's own experiments were implemented in Python; keeping the scheduler in Python keeps us close to the reference implementation for fidelity.
- *Q: Why not microservices?* — A: Explicit project scope constraint (NFR-MAINT-2) — this is a bounded academic project, not a system needing independent service scaling.
- *Q: How does the refresh token work?* — A: Short-lived JWT access token plus a longer-lived refresh token; each refresh call rotates the token (old one revoked, new one issued), and if a revoked token is reused, the whole token family is invalidated — standard protection against stolen-token replay.

**On implementation status:**
- *Q: Can you show me it running?* — Answer honestly per §2.7. Don't claim a full working demo if you don't have one — offer what you *can* show (algorithm test output, DAG parsing, etc.).
- *Q: Have you reproduced the paper's own results?* — A: Yes — the scheduler correctly reproduces the parent/children extraction and the qualitative trend of the paper's α-sensitivity table; full numeric reproduction of Table 3 is [in progress / done — say whichever is true].

**On novelty / conference-worthiness:** see §4 — don't wing this one.

---

## 4. The Question You Most Need To Be Ready For

**"What's actually new here — you're just implementing someone else's algorithm?"**

This is the single most likely hard question, and right now your project documents don't give you a fully solid answer — the "CCOA" multi-cloud/explainability angle is discussed in your Developer Playbook but **isn't in your actual SRS**, meaning it isn't formally specified or (as far as your documents show) built. If you claim it verbally without it being in your spec, that's a gap a guide will catch immediately.

**Pick one honest angle tonight and commit to it:**

1. **Replication + platformization angle** (safest, fully supported by what you actually have): *"Our contribution is taking a paper that was validated only as a standalone Python script and turning it into a configurable, interactive, multi-algorithm comparison platform — with a live α control, persisted run history, and side-by-side comparison against three baselines, none of which the original paper offers as a usable tool."* This is true today and defensible.

2. **Extension angle** (stronger for a conference, but only claim it if you actually build it first): formally add multi-cloud cost comparison / explainability / budget constraints to your SRS as new FR-* items, build at least a minimal working version, *then* claim it as your novelty. Don't claim it before it exists in both the spec and the code.

3. **Empirical angle**: run all 4 algorithms on your *own* generated task sets (not just citing the paper's numbers) and show your own comparative results. Independent empirical validation is a legitimate, modest research contribution on its own.

**Recommended for tomorrow:** lead with #1 (true today), mention #2 as "planned extension for the conference submission" (honest framing of future work), and if you have time before the conference deadline, actually build a minimal slice of #2 or #3 to strengthen the submission.

---

## 5. Glossary — Have These Definitions Ready Cold

| Term | Definition |
|---|---|
| DAG | Directed Acyclic Graph — nodes are tasks, edges are dependencies, no cycles allowed |
| VM | Virtual Machine — a simulated compute unit with a fixed MIPS rating |
| MIPS | Million Instructions Per Second — a VM's processing capacity |
| MI | Million Instructions — the workload size of a task |
| ETC | Expected Time to Compute = MI ÷ MIPS, for a task on a given VM |
| Makespan | Total time to complete all tasks in a workflow |
| Idleness Cost | Cost incurred by faster/costlier VMs sitting idle while others finish |
| α (alpha) | Tunable factor [0,1] trading off completion time against cost in cost-aware mode |
| DAX | Pegasus DAX XML format — a standard representation for scientific workflows |
| JWT | JSON Web Token — stateless session authentication |

---

## 6. The Three Baseline Algorithms — Know How Each Differs

Your SRS (FR-SCH-6) requires all four algorithms implemented and directly comparable. Be ready to explain each in one line:

| Algorithm | Selection Rule | Handles Dependencies? | Weakness |
|---|---|---|---|
| **SJF** (Shortest Job First) | Picks the smallest task first | No native support | Starves large tasks; ignores dependency order entirely |
| **Classical Min–Min** | Among all ready tasks, picks the one with the *minimum* completion time first | No native support | Favors small/fast tasks, can delay large critical-path tasks |
| **Classical Max–Min** | Among all ready tasks, picks the one with the *maximum* completion time first, runs it on the fastest available VM | No native support | This is the base algorithm the paper improves on |
| **Advanced Cost-Aware Max–Min** (yours) | Picks max(own exec time + children's exec time) among ready tasks; VM chosen by speed or α-weighted cost | Yes — respects DAG edges | O(mn²), doesn't exhaustively search all task/VM pairs |

**One-line answer if asked "why compare against these specific three":** they're the paper's own baselines — using the same comparison set lets your results be checked directly against the published ones.

---

## 7. Security — Q&A from SRS §5.2

| Q | A |
|---|---|
| How do you prevent SQL injection? | Parameterized queries / ORM only, never string-concatenated SQL. |
| What's XXE and how do you prevent it? | XML External Entity attacks exploit XML parsers that resolve external entities, potentially leaking files or enabling SSRF. We disable external entity resolution in the XML parser used for DAX/XML workflow uploads. |
| How are passwords stored? | bcrypt with a cost factor ≥ 10, never plaintext. |
| How is file upload secured? | Extension whitelist, size limit, content-type verification, storage kept outside the web root. |
| What about brute-force login attempts? | Rate limiting is applied to authentication and scheduler endpoints. |
| What's your CORS policy? | Restricted to known frontend origin(s) only — no wildcard. |
| How are refresh tokens protected? | Stored as a salted hash only, single-use rotation on every refresh, and reuse of a revoked token invalidates the entire token family. |

---

## 8. Failure Handling — Q&A from SRS §5.3

| Failure | What Happens |
|---|---|
| Invalid XML upload | Rejected with a line-level parse error |
| Circular dependency | Rejected at upload with the specific cycle path shown — never reaches the scheduler |
| Database failure | API returns 500s; health check + retry-with-backoff; connection pooling |
| Scheduler crash | Wrapped in try/except; safe 500 returned; stack trace logged |
| JWT expired | 401 on protected routes; prompts re-login or triggers refresh-token flow |
| No VM configured | Scheduling run blocked with a clear message |
| Missing VM pricing field | Rejected at VM-creation validation, before it can break a cost-mode run later |

**Good line to use if asked "what's your error-handling philosophy":** *"Fail fast and reject invalid input before it ever reaches the scheduler — nothing partially-completed or silently wrong is ever shown to the user as if it succeeded."*

---

## 9. Testing & Validation Strategy — SRS §7.1

| Test Type | Tooling | Focus |
|---|---|---|
| Unit | PyTest (scheduler), Jest (backend) | Algorithm correctness, Eq. 1–3 formula correctness |
| Integration | Supertest + test DB | Full upload → schedule → result round trip |
| API | Postman/Newman | All endpoints and error codes |
| Stress | Locust/k6 | 100+ task DAGs, concurrent users |
| Benchmark | Custom script | Advanced vs. 3 baselines on identical task sets |
| Manual | — | UI flows, edge-case uploads |

**Acceptance criterion to quote if asked "how do you know it's correct":** *"The implementation must reproduce the paper's worked example — 3 VMs at 5000/3000/2000 MIPS — with a makespan of 5.9 seconds, before it's trusted on any user-uploaded workflow."*

---

## 10. Related Work — How Infralytix Sits in the Literature

If your guide is evaluating this for conference-worthiness, expect "what else exists in this space." The source paper's own related-work section surveys these — know 3–4 by name:

| Approach | Core Idea |
|---|---|
| Cui et al. — Markov-based | Multi-objective scheduling via queuing theory + genetic algorithm |
| Liu et al. — Genetic Algorithm | Deadline-constrained scientific workflow scheduling, beats PSO on cost/deadlines |
| Sreenu & Sreelatha — W-Scheduler | Whale Optimization Algorithm for multi-objective task allocation |
| Kumar & Kousalya — Crow Search | Bio-inspired search minimizing makespan |
| Brar & Rao — Workflow Max–Min | **This is the direct baseline your paper improves on** — handles dependencies but executes independent applications sequentially, not concurrently |

**One-line positioning statement:** *"Most prior work is either purely heuristic/metaheuristic (GA, PSO, WOA, Crow Search) without a clean cost-aware mechanism, or cost-aware without dependency handling. This paper's contribution — and by extension ours — is combining dependency-aware scheduling with a tunable cost/speed tradeoff in one simple, non-metaheuristic algorithm."*

---

## 11. What a Conference Paper Actually Needs

Since you're applying for a conference, not just presenting to your guide, be ready to show you understand the target format, not just the project:

- **Abstract** — problem, approach, key result, in ~150 words
- **Related Work** — §10 above is your starting point
- **Methodology** — the algorithm (§2.3) + your platform architecture (§2.5)
- **Results** — this is your current weak point (§2.7) — you need actual run data, even from a small self-generated task set, not just citing the paper's own numbers
- **Conclusion / Future Work** — the CCOA extension angle (§4, option 2) belongs here as honest future work, not as a current claim

If your guide asks "is this ready for submission," the honest answer today is likely "the methodology and platform are solid; I need results data before it's submission-ready" — that's a fine thing to say.

---

## 12. Two-Minute Pre-Meeting Checklist

- [ ] Can you say the 60-second pitch (§1) without notes?
- [ ] Can you write Eq. 1–3 from memory?
- [ ] Have you picked and rehearsed your novelty angle (§4)?
- [ ] Do you know which of the 7 mismatches (§0) you actually fixed vs. are just prepared to explain verbally?
- [ ] Can you name 2–3 related-work approaches (§10) without reading them?
- [ ] Do you have an honest one-sentence answer for "what's actually running right now"?

---

## 13. One-Page Cheat Sheet

| Fact | Value |
|---|---|
| Paper | Raeisi-Varzaneh et al., Cluster Computing, 2024 |
| Algorithm name | Advanced Cost-Aware Max–Min |
| Key formulas | MS = max(MS(VM_i)) · U = (ΣMS(VM_i)/MS)×(1/k) · IC = Σ(MS−MS(VM_i))×Cost(VM_i) |
| Task selection rule | max(own exec time + children's exec time), among ready (parent-less) tasks |
| VM selection (cost-aware) | minimize α×NormalizedTime + (1−α)×NormalizedCost |
| Time complexity | O(mn²) — same order as classical Max–Min |
| Paper's test setup | 3 VMs, 5000/3000/2000 MIPS, costs 15/10/5 per 1000 MI |
| Paper's headline results | waiting time 5.27s→4.67s; utility +34%; idleness cost −92% (122→10) |
| Worked-example makespan | 5.9s |
| Your 4 algorithms | Advanced, classical Max–Min, Min–Min, SJF |
| Stack | React / Node+Express / Python+NetworkX / MySQL / JWT+bcrypt |
| Architecture style | Layered, single logical backend (not microservices) |
| Your novelty (tonight's honest version) | Platformization + multi-algorithm comparison + live α control |
