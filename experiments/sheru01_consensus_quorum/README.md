# Consensus quorum experiment

I chose the built-in `consensus` scenario because it makes a governance
tradeoff visible: how much agreement should a group of autonomous agents need
before acting? This connects to my interest in accountable delegation among
autonomous agents.

## One-variable change

I copied the built-in scenario and changed exactly one supported setting:

```diff
-    quorum: 0.667
+    quorum: 0.80
```

No failure injection, seed, agent, layer, duration, or other task setting
changed. A direct diff against `scenarios/consensus.yaml` therefore contains
one changed line.

## Pre-run hypothesis

With 19 followers, the change raises the effective threshold from 13 to 16
accept votes. Before running the changed scenario, I expected the higher
threshold to make commitment harder and possibly require more proposal rounds.
Because I did not inject failures, I expected message delivery to stay at 100%.

## Method

I followed `docs/quickstart.md`, installed `nest-core[plugins]`, and confirmed
`nest doctor` reported `7/7 checks passed`. I used NEST 0.1.4, Python 3.14.0,
seed 42, and the repository at commit
`87ed39313d4678908708282a1ca32e11831271c6`.

```bash
nest run consensus \
  -o experiments/sheru01_consensus_quorum/evidence/baseline.jsonl
nest run experiments/sheru01_consensus_quorum/consensus_quorum_80.yaml \
  -o experiments/sheru01_consensus_quorum/evidence/quorum-80.jsonl
nest inspect experiments/sheru01_consensus_quorum/evidence/baseline.jsonl
nest inspect experiments/sheru01_consensus_quorum/evidence/quorum-80.jsonl
```

I also ran the built-in consensus validators against both traces and repeated
both runs. Each repeat was byte-identical to its original trace.

Artifacts: [baseline trace](evidence/baseline.jsonl),
[0.80-quorum trace](evidence/quorum-80.jsonl),
[baseline report](evidence/baseline-report.html), and
[0.80-quorum report](evidence/quorum-80-report.html).

## Evidence

| Observation | Baseline: 0.667 | Experiment: 0.80 |
|---|---:|---:|
| Effective threshold | 13/19 | 16/19 |
| Proposal rounds attempted | 3 | 5 |
| Accept tallies by round | 12, 11, 13 | 12, 11, 13, 10, 15 |
| Outcome | committed in round 3 | all five rounds aborted |
| Message count (send + receive) | 342 | 570 |
| Delivery / `success_rate` | 1.000 / 1.000 | 1.000 / 1.000 |
| Dropped messages | 0 | 0 |

The first three vote tallies were identical because the seed and all other
settings were unchanged. The higher quorum turned the baseline's round-3
commit at 13/19 into an abort and exhausted all five configured rounds. Two
extra attempts increased message traffic by 66.7%, but produced no decision.

## Validator output and investigation

All three validators passed for both traces. The important detail was their
reported coverage:

```text
baseline:   PASS consensus_agreement - checked 1 committed rounds
experiment: PASS consensus_agreement - checked 0 committed rounds
experiment: PASS consensus_validity - all 0 committed values were proposed
experiment: PASS consensus_no_conflict - checked 0 rounds
```

I was surprised that the experimental run reported `success_rate: 1.000` and
passed every validator despite committing no value. I checked the raw trace,
the consensus agent, the metric implementation, and the validators. The trace
contains five `result:*:aborted:*` outcomes and no commit. `success_rate` is a
backward-compatible alias for message delivery, so it measures transport, not
consensus progress. The validators enforce safety properties only for committed
rounds; zero commits pass vacuously. This is a measurement gap, not a hidden
successful decision.

## What I would build next

I would add a consensus-liveness evaluator that reports decision rate,
attempts-to-decision, and no-decision exhaustion, with separate safety and
liveness summaries. A governance system should distinguish reliable message
delivery from the group's ability to authorize an action.

## AI and other help used

This work was executed through OpenAI Codex in my local workspace. Codex helped
navigate the unfamiliar repository, record the pre-run hypothesis, execute the
documented CLI workflow, inspect and compare traces, read the validator,
metric, and agent code, repeat the runs, draft this README, and prepare the pull
request. Other tools were the repository's quickstart and scenario guide,
Python, NEST CLI, Git, and the connected GitHub account. No additional human
help was used.

## Verification

- `nest doctor`: 7/7 checks passed
- Direct scenario diff: only `task.config.quorum` changed
- Baseline and experiment reruns: byte-identical traces
- Ruff lint and format checks: passed
- Pyright strict type check: 0 errors
- Pytest: 1,311 passed, 1 skipped, 1 deselected
