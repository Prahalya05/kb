# 01 · Evals: Turning "It Seems Better" Into Numbers

> Level: Pro · Time: 4 hours · Prereqs: all intermediate and advanced lessons; basic probability (binomial coefficients)
> Code is stdlib-only Python 3.13. Guidance from Anthropic docs/engineering posts stamped with dates.

---

## 1. Why this lesson exists

Every lesson so far ended with "measure it". This one shows how. LLM systems are **stochastic** (lesson 01), **sensitive to small prompt changes** (lesson 02), and **multi-step** (agents), so intuition about whether a change helped is unreliable. The METR study (01-beginner/03) is a sobering example of perception diverging from measurement: developers *believed* they were 20% faster while being 19% slower.

Anthropic's prompt-engineering guide assumes you have success criteria and "some ways to empirically test against those criteria" before you even start, and its January 2026 post "Demystifying evals for AI agents" makes evals the backbone of agent development. In industry, the ability to design an eval suite is one of the clearest differences between someone who *uses* AI tools and someone who *engineers* with them.

## 2. Intuition

An eval suite is a **unit-test suite for behaviour that is probabilistic**. Two twists versus normal tests:

1. A test can pass on one run and fail on the next, so you run each task several times and report **rates**, not booleans.
2. Many outputs have no single correct string ("write a helpful reply"), so some graders must be judgement-based (a rubric applied by a model or a human).

Think of it like a cricket selector judging a batter: not one innings, but an average over many, across different pitches (task types), and with attention to *consistency* (does he score every match, or one century and nine ducks?).

## 3. Precise definitions

From Anthropic's "Demystifying evals for AI agents" (Jan 9, 2026):

- **Task**: "a single test with defined inputs and success criteria".
- **Trial**: "Each attempt at a task is a trial."
- **Grader**: "logic that scores some aspect of the agent's performance".
- **Transcript / trajectory**: "the complete record of a trial", including tool calls and intermediate results.
- **Outcome**: "the final state in the environment at the end of the trial".
- **Evaluation harness**: "the infrastructure that runs evals end-to-end"; **agent harness**: "the system that enables a model to act as an agent".
- **pass@k**: "the likelihood that an agent gets at least one correct solution in k attempts".
- **pass^k**: "the probability that all k trials succeed".
- **Capability evals** start at a low pass rate (what can it do?); **regression evals** "should have a nearly 100% pass rate" (does it still do what it did?).

Grader types (same post, and Anthropic's "Define success and build evaluations" docs):

| Grader | Strengths | Weaknesses |
|---|---|---|
| Code-based (exact match, string match, tests pass, state checks) | Fast, cheap, objective, reproducible | Brittle to valid variations, little nuance |
| Model-based (LLM-as-judge with rubric) | Flexible, scalable, handles nuance | Non-deterministic, costs money, needs calibration against humans |
| Human | Gold standard, calibrates other graders | Slow, expensive, hard to scale |

The docs' ordering advice: "choose the fastest, most reliable, most scalable method", and "Prioritize volume over quality: More questions with slightly lower signal automated grading is better than fewer questions with high-quality human hand-graded evals."

## 4. Step by step

### 4.1 Dumbest approach: try three prompts, eyeball the outputs

Walls: three samples tell you almost nothing about a stochastic system; you remember the striking failure and forget the quiet successes; next week's model update silently changes behaviour and nobody notices.

### 4.2 Next: a fixed task set with code graders

Collect tasks **from real failures** ("20-50 simple tasks drawn from real failures is a great start", per the post), write graders that check the **outcome**, run once per task. Wall: one run per task is a coin flip for tasks the agent solves 60% of the time.

### 4.3 Next: multiple trials and the right metric

Run n trials per task. Now choose a metric that matches the product:

- A coding assistant where a human picks the best of 3 suggestions: **pass@3** (at least one good).
- An autonomous agent that must succeed every time it runs (a nightly job): **pass^k** (all k succeed). This is the metric that exposes flakiness.

**Estimating pass@k without bias.** If you have n trials with c successes, naively computing 1 − (1 − c/n)^k is biased. Chen et al. (2021, the Codex/HumanEval paper) give the unbiased estimator pass@k = 1 − C(n−c, k) / C(n, k): the probability that a random size-k subset of your n trials contains at least one success. Analogously, pass^k = C(c, k) / C(n, k): the probability a random size-k subset is all successes.

### 4.4 Next: uncertainty and gates

A suite score of 0.68 on 4 tasks is noise. Report a **confidence interval**: the bootstrap resamples tasks with replacement and recomputes the mean many times. Then put a **regression gate** in CI: fail the build if the score drops more than a tolerance below the last release.

### 4.5 The harness

```python
# eval_harness.py - a minimal eval harness: tasks, trials, code graders, pass@k, pass^k, CI gate.
# Pure Python 3 standard library. The "agent" is a stub with a known per-task success rate.
import math                         # comb() for the unbiased pass@k estimator
import random                       # stub randomness + bootstrap resampling
import statistics                   # mean

rng = random.Random(7)

# A task = input + grader. Graders check the OUTCOME, not the path the agent took.
TASKS = [
    {"id": "sum-list", "input": [3, 4, 5], "grade": lambda out: out == 12},
    {"id": "max-list", "input": [3, 9, 5], "grade": lambda out: out == 9},
    {"id": "empty-sum", "input": [], "grade": lambda out: out == 0},
    {"id": "neg-max", "input": [-3, -9], "grade": lambda out: out == -3},
]
SUCCESS_RATE = {"sum-list": 0.95, "max-list": 0.8, "empty-sum": 0.5, "neg-max": 0.3}


def stub_agent(task: dict) -> int | None:
    """Pretend agent: right answer with the task's success rate, else garbage."""
    truth = {"sum-list": 12, "max-list": 9, "empty-sum": 0, "neg-max": -3}[task["id"]]
    return truth if rng.random() < SUCCESS_RATE[task["id"]] else None


def pass_at_k(n: int, c: int, k: int) -> float:
    """Unbiased estimator from Chen et al. (2021): 1 - C(n-c, k) / C(n, k)."""
    if n - c < k:
        return 1.0
    return 1.0 - math.comb(n - c, k) / math.comb(n, k)


def pass_hat_k(n: int, c: int, k: int) -> float:
    """pass^k: probability that ALL k sampled trials succeed = C(c, k) / C(n, k)."""
    return math.comb(c, k) / math.comb(n, k) if c >= k else 0.0


def bootstrap_ci(xs: list[float], iters: int = 2000) -> tuple[float, float]:
    """95% percentile bootstrap interval for the mean: resample tasks with replacement."""
    means = sorted(statistics.mean(rng.choices(xs, k=len(xs))) for _ in range(iters))
    return means[int(0.025 * iters)], means[int(0.975 * iters)]


N_TRIALS, K = 10, 3
rows = []
for t in TASKS:
    c = sum(t["grade"](stub_agent(t)) for _ in range(N_TRIALS))   # c = successes out of n trials
    rows.append((t["id"], c, pass_at_k(N_TRIALS, c, 1), pass_at_k(N_TRIALS, c, K), pass_hat_k(N_TRIALS, c, K)))
    print(f"{t['id']:10} {c:2}/{N_TRIALS}  pass@1={rows[-1][2]:.2f}  pass@{K}={rows[-1][3]:.2f}  pass^{K}={rows[-1][4]:.2f}")

p1 = [r[2] for r in rows]
lo, hi = bootstrap_ci(p1)
print(f"suite pass@1 = {statistics.mean(p1):.2f}  (95% CI {lo:.2f}-{hi:.2f}, only {len(TASKS)} tasks!)")

BASELINE = 0.70                                                     # last release's score
if statistics.mean(p1) < BASELINE - 0.05:
    print("CI GATE: FAIL - regression beyond tolerance")
else:
    print("CI GATE: pass")
```

Output:

```text
sum-list   10/10  pass@1=1.00  pass@3=1.00  pass^3=1.00
max-list    8/10  pass@1=0.80  pass@3=1.00  pass^3=0.47
empty-sum   6/10  pass@1=0.60  pass@3=0.97  pass^3=0.17
neg-max     3/10  pass@1=0.30  pass@3=0.71  pass^3=0.01
suite pass@1 = 0.68  (95% CI 0.43-0.90, only 4 tasks!)
CI GATE: pass
```

**Line-by-line walkthrough (syntax and logic):**

- `TASKS = [{"id": ..., "input": ..., "grade": lambda out: out == 12}, ...]`: each task carries its own grader as a lambda (an inline function). Graders check the **outcome** (the returned value), not how the agent got there, following the post's advice that "it's often better to grade what the agent produced, not the path it took".
- `SUCCESS_RATE`: the stub's true per-task success probabilities, so we can see whether the metrics recover them.
- `stub_agent`: returns the right answer with that probability, else `None`. `int | None` in the return hint means "int or None".
- `pass_at_k(n, c, k)`: `math.comb(a, b)` is the binomial coefficient "a choose b". If fewer than k failures exist (`n - c < k`), every k-subset must contain a success, so return 1.0.
- `pass_hat_k(n, c, k)`: `C(c,k)/C(n,k)`, zero when there are fewer than k successes.
- `bootstrap_ci`: `rng.choices(xs, k=len(xs))` samples tasks **with replacement**; repeating 2000 times and taking the 2.5th and 97.5th percentiles of the sorted means gives a 95% percentile interval.
- `c = sum(t["grade"](stub_agent(t)) for _ in range(N_TRIALS))`: run 10 trials, grade each, count successes (`True` is 1).
- The formatted print uses `{x:.2f}` (two decimals) and `{name:10}` (pad to width 10).
- The gate: compare the suite mean against `BASELINE - 0.05`.

**What the output teaches:**

- `max-list`: pass@1 0.80 but **pass^3 0.47**. A "usually works" agent fails at least once in three runs more than half the time. For autonomous jobs, pass^k is the honest number.
- `neg-max` (negative numbers): 0.30 pass@1, an edge case that a happy-path demo would never reveal.
- The suite's 95% CI is **0.43–0.90** with only 4 tasks: the point estimate 0.68 is nearly meaningless. More tasks, not more decimals.
- The CI gate passed at 0.68 vs a 0.70 baseline because of the 0.05 tolerance; with this much uncertainty, a tighter tolerance would make the gate flaky. That is a real design tension.

### 4.6 Model-based graders done carefully

When outputs are open-ended, use an LLM judge, but treat it as a measuring instrument that must be **calibrated**:

- Give it a **detailed rubric** and a constrained verdict ("output only 'correct' or 'incorrect'", or a 1–5 scale), per Anthropic's docs; let it reason before scoring ("use a grader model with thinking on").
- Judge each dimension separately (correctness, tone, citation accuracy) rather than one holistic score.
- Measure agreement with human labels on a sample before trusting it at scale.
- Know the documented biases: Zheng et al. (2023), "Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena", examined position bias, verbosity bias and self-enhancement bias. Mitigations include swapping answer order and averaging.
- Never let the system under test grade itself with the same prompt.

### 4.7 Agent evals specifics

```mermaid
flowchart LR
  T[Task: input + environment setup] --> H[Eval harness]
  H -->|n trials| A[Agent harness + model]
  A --> TR[(Transcripts)]
  A --> OUT[(Outcome state)]
  OUT --> CG[Code graders: tests pass? file exists? DB row correct?]
  TR --> MG[Model graders: rubric on reasoning/communication]
  CG --> M[pass@k, pass^k, CI, cost, latency]
  MG --> M
  M --> G{Regression gate}
```

- Grade the **final environment state** (tests pass, the right file changed, no forbidden file touched), not a specific sequence of tool calls.
- Track **cost and latency per task** alongside success; an agent that succeeds by using 10× more tokens is a regression too.
- **Read the transcripts.** The post is emphatic: "You won't know if your graders are working well unless you read the transcripts and grades from many trials." Graders have bugs; transcripts reveal them (e.g. a grader that passes because the agent deleted the failing test).
- Isolate trials: each trial must start from a clean environment, or state leaks between trials inflate scores.

### 4.8 Benchmarks vs your evals

Public benchmarks (e.g. SWE-bench, Jimenez et al. 2023, which tests resolving real GitHub issues) are useful for comparing models in general, but they are not your task distribution, and widely published benchmarks risk contamination (appearing in training data). Your own eval set, drawn from your real failures, is what tells you whether *your* system improved.

## 5. Common mistakes

1. **Single runs per task.** Report rates over multiple trials.
2. **pass@k for autonomous jobs.** Use pass^k (or both).
3. **Grading the path** instead of the outcome; valid alternative solutions fail.
4. **Unvalidated LLM judges.** Calibrate against humans; check position and verbosity bias.
5. **Tiny suites with precise-looking scores.** Show confidence intervals.
6. **Never reading transcripts.** Grader bugs go unnoticed.
7. **Shared state between trials.**
8. **Optimising the eval instead of the product** (Goodhart's law): keep adding fresh tasks from new real failures.

## 6. Connections to other tracks

- **Testing/Verification**: evals are tests for stochastic systems; regression gates are CI; graders are oracles; differential testing (next lesson) is a grader strategy.
- **Math for CS**: binomial coefficients, unbiased estimators, bootstrap confidence intervals.
- **Research skills**: experimental design, held-out sets, reporting uncertainty, avoiding p-hacking via many prompt variants.
- **Cloud/DevOps**: running eval suites in CI on every prompt/model change.

## 7. Where this shows up in my projects

- **compiler-fuzz** already *is* an evaluation harness in the classical sense: generated programs are tasks, GCC vs Clang at -O0/-O2/-O3 is a differential grader, and the planted-bug validation measures the harness's own detection rate. That last idea is exactly "calibrate your grader": you injected known bugs to check the checker.
- **QiskitSage**: an eval set of past Qiskit PRs with known issues lets you compute per-agent precision/recall of findings and pass^k for "finds the known bug every run", turning "the reviewer seems good" into a number that guards against regressions when you change prompts or local models.

## 8. Self-check questions

1. Why is pass^k the right metric for an unattended nightly agent?
2. With n=10 trials and c=8 successes, compute pass@3 and pass^3 using the estimators.
3. Why grade outcomes rather than tool-call sequences?
4. What does a 95% bootstrap CI of 0.43–0.90 tell you about a suite?
5. Name two biases of LLM judges and one mitigation for each.

<details><summary>Answers</summary>

1. The job must succeed every run; pass^k measures the probability that all k runs succeed, exposing flakiness that pass@k hides.
2. pass@3 = 1 − C(2,3)/C(10,3) = 1 − 0 = 1.00 (only 2 failures, fewer than 3). pass^3 = C(8,3)/C(10,3) = 56/120 ≈ 0.47. These match the `max-list` row.
3. Many valid paths exist; grading the path penalises correct alternative solutions and rewards brittle imitation. The outcome is what users care about.
4. The suite is too small (or too heterogeneous) for the point estimate to be reliable; differences smaller than the interval are not meaningful. Add tasks.
5. Position bias: swap the order of compared answers and average. Verbosity bias: rubric penalising unnecessary length, or length-controlled comparisons. (Self-enhancement bias: use a different model or calibrate against human labels.)

</details>

## 9. Mnemonic (Tamil + English)

**"Oru dhadava jeyichaa pass@k, ellaa dhadavaiyum jeyichaa pass^k."**
(ஒரு தடவ ஜெயிச்சா pass@k, எல்லா தடவையும் ஜெயிச்சா pass^k: win once → pass@k; win every time → pass^k.)

**"Transcript-a padikkaama, score-a nambaadhe."** (டிரான்ஸ்கிரிப்ட்-அ படிக்காம, ஸ்கோர்-அ நம்பாதே: don't trust a score whose transcripts you haven't read.)

## Sources

- Anthropic, "Demystifying evals for AI agents" (Jan 9, 2026): https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents
- Anthropic, Define success and build evaluations (as of Nov 2026): https://platform.claude.com/docs/en/test-and-evaluate/develop-tests
- Chen et al., "Evaluating Large Language Models Trained on Code" (2021), unbiased pass@k: https://arxiv.org/abs/2107.03374
- Zheng et al., "Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena" (2023): https://arxiv.org/abs/2306.05685
- Jimenez et al., "SWE-bench: Can Language Models Resolve Real-World GitHub Issues?" (2023): https://arxiv.org/abs/2310.06770
- Efron & Tibshirani, *An Introduction to the Bootstrap* (1993), the standard bootstrap reference (book).
- Becker et al. (METR), "Measuring the Impact of Early-2025 AI on Experienced Open-Source Developer Productivity" (2025): https://arxiv.org/abs/2507.09089

## Verify yourself

1. Run `python3 eval_harness.py`. Raise `N_TRIALS` to 100 and watch pass@1 converge to the true rates in `SUCCESS_RATE`.
2. Add 20 more tasks with random success rates and observe how the CI narrows.
3. Check by hand: for n=10, c=3, k=3, pass@3 = 1 − C(7,3)/C(10,3) = 1 − 35/120 ≈ 0.71 (the `neg-max` row).
4. Read Section 2.1 of Chen et al. (2021) for the derivation of the estimator and its numerically stable implementation.
5. Read the "Demystifying evals" post and list which grader types it recommends for coding agents.
