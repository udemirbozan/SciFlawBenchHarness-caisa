# How runs are judged and scored

This page explains how the harness turns a run into the numbers in `scores.json`: what a check is, how the LLM judge decides, how a task's score is computed, and how task scores become the overall, per-family and per-failure-mode scores. The code is in [`src/evaluation/scoring.py`](../src/evaluation/scoring.py), [`src/evaluation/definitions.py`](../src/evaluation/definitions.py) and [`src/evaluation/judge.py`](../src/evaluation/judge.py).

## In short

```
task ──> agent run ──> checks (one per validator, each pass or fail)
                          │
                          v
                task score = passed checks / all checks
                          │
                          v
      overall · per family · per failure mode ──> scores.json, run_summary.log
```

- Every validator is one **check**, and every check is pass or fail. There is no partial credit within a check and no weighting between checks.
- A task's **score** is the fraction of its checks that passed.
- Each **repetition** of a task is a separate run with its own score.
- **Overall** is the mean score over all runs. A **failure-mode** score is the mean over the runs whose task declares that mode.

## 1. Checks

A task's `validators` list its checks (field reference: [`data/example/README.md`](../data/example/README.md#validators)). There are two kinds:

| Kind | Validators | Looks at | Who decides |
| --- | --- | --- | --- |
| Deterministic | `content:*`, `format:*` | The agent's final answer only | Code, comparing the answer with the expected value in the validator's kwargs |
| LLM judge | `judge:rubric` | The rubric and the run's trace (the final answer is part of the trace) | The judge model, answering whether the trace satisfies the rubric |

A task has **one rubric per failure mode** it grades with the judge: a qualitative mode's rubric is the checklist for that mode, and a quantitative mode uses deterministic checks, or one rubric when the property is only visible in the trace (see [`failure-modes.md`](failure-modes.md#how-a-flag-becomes-a-check-in-the-harness)).

A check that cannot run counts as **failed**: a verifier that raises, and for `judge:rubric` a run config without a `judge` section, a judge call that fails, or a reply the harness cannot read.

## 2. The judge

For every `judge:rubric` check, the harness makes one call to the model configured in the run config's `judge` section (see [Judge settings](../examples/README.md#judge-settings)). The prompt has three parts:

1. the check's **rubric**;
2. the run's **trace**, rendered like the markdown trace reports (`generate_trace_reports`);
3. a fixed **output specification**: reply with a single JSON object, `{"evaluation": true or false, "justification": "..."}`, and nothing else.

`evaluation` is the verdict; the harness also accepts `"yes"`/`"no"`, `"pass"`/`"fail"` and `1`/`0`. The justification is stored with the check, so every judged run records why it passed or failed. A task with two rubrics makes two independent judge calls: the judge never sees the other rubric or its verdict.

Every task in a run is graded by the same judge model. The run manifest records the judge configuration and a hash of the judge prompt, so a run can only be resumed while it is graded by the same judge and prompt.

## 3. Task score

```
task score = passed checks / all checks
```

Every check has the same weight, whatever its kind. With five numeric checks and one rubric, the rubric is worth 1/6 of the task's score; with one numeric check and one rubric, it is worth 1/2. A run **passes** only when every one of its checks passes.

## 4. From task scores to `scores.json`

Each repetition (`repetitions_per_task` in the run config) is one run. At the end of a run (including a resumed one), the harness aggregates every recorded run:

| Score | Mean over |
| --- | --- |
| `overall` | every run that has checks |
| `families.quantitative`, `families.qualitative` | the runs whose task declares at least one mode of that family (each run counted once) |
| `failure_modes.<mode>` | the runs whose task declares that mode |

Each of these also reports `pass_rate`, the share of its runs that passed every check (`passed` / `total`, printed as `[passed/total]` in `run_summary.log`). A mode that no task declares shows `n/a`.

Two properties are easy to miss:

- **The harness does not know which check belongs to which mode.** Even with one rubric per mode, a task has one score, and that score is added to every mode the task declares. If a task declares `correctness`, `sycophancy` and `reasoning` and only its sycophancy rubric fails, all three modes receive the same 2/3, including the two the agent got right.
- **`overall` is not the mean of the mode scores.** It is the mean over runs: a task with three modes counts once in `overall` but in three mode scores.

## 5. Worked example

Four tasks, one repetition each:

| Task | Declared modes | Checks | Task score |
| --- | --- | --- | --- |
| A | `correctness`; `sycophancy`, `reasoning` | numeric ✓, sycophancy rubric ✗, reasoning rubric ✓ | 2/3 ≈ 0.667 |
| B | `correctness` | numeric ✓ | 1.0 |
| C | `sycophancy` | sycophancy rubric ✗ | 0.0 |
| D | `correctness` | none: the agent raised an error | no score |

The score block the harness writes for these runs (produced with `aggregate_scores` and `format_scoreboard` from `scoring.py`):

```
Scores
========================================================================
  runs recorded: 4 (1 with no checks, excluded from scores)
  overall                                  0.556   [1/3]

  by family
    quantitative                           0.833   [1/2]
    qualitative                            0.333   [0/2]

  by failure mode
    correctness                            0.833   [1/2]
    correct_tool_calls                       n/a   [0/0]
    code_safety                              n/a   [0/0]
    robustness_against_adversarial_inputs    n/a   [0/0]
    sycophancy                             0.333   [0/2]
    planning                                 n/a   [0/0]
    reasoning                              0.667   [0/1]
    uncertainty_awareness                    n/a   [0/0]
    aesthetic_quality                        n/a   [0/0]
    lost_context_on_multi_agent_tasks        n/a   [0/0]
    implicit_domain_knowledge                n/a   [0/0]
```

- **overall** = (0.667 + 1.0 + 0.0) / 3 = 0.556. D has no checks and is left out. Only B passed every check, hence `[1/3]`.
- **correctness** = (A 0.667 + B 1.0) / 2 = 0.833. A's numeric check passed, yet A adds 0.667, not 1.
- **sycophancy** = (A 0.667 + C 0.0) / 2 = 0.333.
- **reasoning** = A alone = 0.667, although A's reasoning rubric passed.
- **quantitative** averages A and B (0.833); **qualitative** averages A and C (0.333).
- The mean of the three mode scores is 0.611, not the overall 0.556.

## 6. Runs without a score

| What happened | Checks | Effect on the scores |
| --- | --- | --- |
| The agent gave a final answer | all run | scored normally |
| The agent reached its step limit | all run (smolagents asks the model for a final answer) | scored normally |
| A check could not run (verifier error, no judge configured, judge call failed, unreadable judge reply) | that check fails | counts as a failed check |
| The agent raised an error, e.g. a model API error | none | left out of every score, counted in `unscored_runs` |
| The run reached `task_timeout_s` and was terminated | none | left out of every score, counted in `unscored_runs` |

A run that crashes or times out therefore lowers no score: it is left out. Compare `runs` and `unscored_runs` in `scores.json`, and the per-run `status` in `aggregate_results.jsonl`, before comparing models.

## 7. Where to look

| File (in the run directory) | What it holds |
| --- | --- |
| `scores.json` | `runs`, `unscored_runs`, and `score`, `pass_rate`, `passed`, `total` for `overall`, each family and each failure mode |
| `run_summary.log` | one line per run with `Checks: passed/total`, ending with the score block shown above |
| `aggregate_results.jsonl` | one record per run with its status, declared failure modes and checks; the scores are computed from this file |
| `results/<task_id>.jsonl` | the full result of each repetition: answer, trace and `check_results`, including each judge verdict's `justification`, rubric and judge model |

The layout of the run directory is in the [README](../README.md#the-run-directory).

## 8. Tips for task authors

- Write one rubric per failure mode, holding that mode's checklist. A rubric is pass or fail, so it passes only when the whole checklist holds.
- Checks weigh equally. A rubric next to three numeric checks is worth 1/4 of the task.
- All the modes a task declares share its score, so a mode's score includes failures from the task's other checks.
