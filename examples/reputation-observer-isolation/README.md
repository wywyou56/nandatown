# Reputation under observer isolation: scoring validator counts sent reports, not delivered ones

MIT AI Studio Fall 2026 NANDA Town assessment
Author: You Wu [@wywyou56](https://github.com/wywyou56)

## Summary

I ran the built-in `reputation` scenario, then changed one supported failure
setting: `failures.network_partition`. The variant places `observer-0` in one
partition and all 20 traders in another.

The partition left every trader-generated send unchanged, but all 80
`report:` messages were dropped before reaching the observer. The observer
therefore issued no warnings. Despite that complete loss of delivered
testimony, `reputation_scoring` returned exactly the same verdict and detail as
the baseline because it evaluates reports when they are **sent**, not when
they are received. `reputation_warnings` detected the outage at seed 42, but a
supplementary paired check found three of ten seeds where both validators
passed while the observer received zero reports.

The result is a narrow evidence-boundary finding: emitted testimony and
delivered accountability are not the same property.

## Scenario and single change

The built-in `reputation` scenario contains:

- 16 honest traders, which always deliver and report cheating;
- 4 malicious traders, which probabilistically deliver or cheat and never
  file reports; and
- 1 observer, which updates scores from reports and broadcasts a warning when
  a score reaches `-3` or below.

The effective flow is:

```text
baseline:  traders  -- report -->  observer  -- warning -->  traders
variant:   traders  -- report -X   observer
           traders  <---------->   traders     (unchanged)
```

The only behavior-affecting setting changed was:

```yaml
failures:
  message_drop: 0.0
  network_partition:
    groups:
      - ["observer-0"]
      - [all 20 trader IDs]
```

The scenario `name` and `description` are labels for the new example; agent
counts, roles, task configuration, layers, duration, seed, and message-drop
rate remain at their built-in values. The complete configuration is
[`reputation-observer-isolated.yaml`](./reputation-observer-isolated.yaml).

## Question

Does `reputation_scoring` establish that testimony reached the observer and
affected accountability, or only that another agent emitted a report?

## Hypothesis, registered before the variant

I ran and inspected the baseline first, then committed
[`Hypothesis.md`](./Hypothesis.md) before executing the partitioned variant.
The commit history therefore records the prediction before the result.

I expected:

1. Trader behavior to remain unchanged because all traders stay connected to
   one another.
2. All 80 `report:` messages to still be sent, but all 80 to be dropped at the
   partition boundary.
3. The observer to receive no inputs and broadcast no warnings, compared with
   three baseline warnings.
4. `reputation_scoring` to return its baseline result unchanged because the
   validator reads sender-side trace events.
5. `reputation_warnings` to fail because sender-side reports still imply that
   three agents need warnings, while the isolated observer emits none.

I marked the first prediction as the main confounding risk: if the partition
changed trading or random choices, an unchanged validator result would be
harder to interpret.

## Method

The primary experiment is one paired comparison using the shipped seed,
`42`, with `nest-core 0.1.4`:

- **Baseline:** built-in `reputation`, unmodified.
- **Variant:** the copied scenario with only
  `failures.network_partition.groups` activated.

Both traces were analysed by event kind and message prefix. Validators were
run with `validate_trace(..., "reputation")`, and their implementations in
[`validators.py`](../../packages/nest-core/nest_core/validators.py) were read
to investigate the results.

## Primary evidence: seed 42

### Emitted behavior stayed fixed; delivery did not

| Trace evidence | Baseline | Observer isolated |
| --- | ---: | ---: |
| Agents | 21 | 21 |
| `send` events | 280 | 280 |
| `trade:` sends | 100 | 100 |
| `deliver:` sends | 87 | 87 |
| `cheat:` sends | 13 | 13 |
| `report:` sends | 80 | 80 |
| `report:` receives at observer | 80 | **0** |
| Dropped `report:` messages | 0 | **80** |
| Observer `warning:` broadcasts | 3 | **0** |
| All `receive` events | 340 | 200 |
| All `dropped` events | 0 | 80 |
| Total trace events | 665 | 602 |

After normalizing each `send` event to `(sender, recipient, payload)`, the
complete 280-event sequences were identical. The raw JSONL files are not
byte-identical: removing the observer's warning broadcasts shifts later
correlation IDs. The controlled claim is therefore that **who sent what to
whom** stayed identical, while delivery to the observer changed.

The baseline contains 60 `warning:` receives because each of the observer's
three broadcasts reaches 20 traders. This also explains why the generic
`success_rate`/`delivery_rate` metric is `1.2143` in the baseline and `0.7143`
in the variant: broadcast receives enter the numerator although broadcasts do
not enter the metric's send denominator. I therefore use explicit event-class
counts rather than treating that metric as protocol success.

### Validator results

| Validator | Baseline | Observer isolated |
| --- | --- | --- |
| `reputation_scoring` | **FAIL** — `cheaters not reported: {'malicious-3'}` | **FAIL** — identical detail |
| `reputation_warnings` | **PASS** — `3 warnings issued, 3 needed` | **FAIL** — malicious-0, malicious-1, and malicious-2 were at/below `-3` but not warned |

All five behavioral predictions held. The byte-level qualification above is
the only caveat: message behavior was identical, but correlation IDs were not.
Most importantly, `reputation_scoring` produced the same verdict and detail
for a functioning observer and an observer that received none of the 80
reports.

Primary traces:

- [`traces/reputation-baseline.jsonl`](./traces/reputation-baseline.jsonl)
- [`traces/reputation-isolated.jsonl`](./traces/reputation-isolated.jsonl)

## Investigation

### Surprise 1: the shipped baseline already fails

Before adding the partition, the built-in scenario failed
`reputation_scoring` at its shipped seed. I counted each malicious agent's
`cheat:` sends and the `report:...:bad` sends naming that agent:

| Agent | Cheats | Bad reports | Unreported cheats | Good reports | Sender-side score |
| --- | ---: | ---: | ---: | ---: | ---: |
| malicious-0 | 4 | 3 | 1 | 0 | -6 |
| malicious-1 | 5 | 4 | 1 | 1 | -7 |
| malicious-2 | 3 | 3 | 0 | 3 | -3 |
| malicious-3 | 1 | 0 | 1 | 4 | **+4** |

Three cheats went unreported. `malicious-3`'s only cheat
targeted `malicious-1`; malicious agents do not report, so nobody testified.
It finished with a score of `+4`, above seven of the sixteen honest agents and
tied with three more.

Reading `validate_reputation_scoring` explained the outcome. It constructs:

- the set of agents named in any sent `cheat:` message; and
- the set of agents named in any sent bad `report:` message.

It subtracts the second set from the first. This checks whether each cheating
**agent** was reported at least once, not whether each cheating **event** was
reported. Thus malicious-0 and malicious-1 pass that coverage check despite
one unreported cheat each, while malicious-3 fails because it has no bad
report at all.

### Surprise 2: one validator caught the outage, but only conditionally

I initially expected both validators to miss the partition because both rely
on emitted evidence. That was wrong for seed 42.

`reputation_warnings` builds scores from emitted `report:` events, then checks
emitted `warning:` events. It therefore still computes malicious-0,
malicious-1, and malicious-2 at or below `-3` in the isolated trace. Because
the observer received no reports and emitted no warnings, the validator
correctly fails.

The check is still conditional on sender-side evidence producing a non-empty
final warning set. If no final score is at or below `-3`, zero warnings can
pass even when the observer received no testimony.

## Supplementary paired-seed check

The seed-42 comparison above is the assessment experiment. Afterward, I ran a
robustness check across ten seeds. Within every pair, the baseline and variant
used the same seed; the partition remained the only treatment difference. The
compact machine-readable results are in
[`seed-sweep-summary.csv`](./seed-sweep-summary.csv). The 20 raw sweep traces
are intentionally omitted from the PR because they repeat the same event
structure and can be regenerated with the commands below.

| Seed | `reputation_scoring` baseline → isolated | `reputation_warnings` baseline → isolated | Baseline warnings | Isolated reports delivered |
| ---: | --- | --- | ---: | ---: |
| 1 | PASS → PASS | PASS → PASS | 0 | 0/80 |
| 2 | PASS → PASS | PASS → FAIL | 2 | 0/80 |
| 3 | PASS → PASS | PASS → FAIL | 2 | 0/80 |
| 7 | PASS → PASS | PASS → FAIL | 2 | 0/80 |
| 11 | PASS → PASS | PASS → FAIL | 3 | 0/80 |
| 42 | FAIL → FAIL | PASS → FAIL | 3 | 0/80 |
| 99 | PASS → PASS | PASS → FAIL | 2 | 0/80 |
| 123 | PASS → PASS | PASS → FAIL | 3 | 0/80 |
| 777 | PASS → PASS | PASS → PASS | 0 | 0/80 |
| 2024 | PASS → PASS | PASS → PASS | 1 | 0/80 |

Across all ten pairs:

- every isolated run sent 80 reports and delivered 0;
- `reputation_scoring` returned the same verdict **and detail** as its baseline
  pair ten times out of ten; and
- the isolated `reputation_warnings` check failed in seven seeds but passed in
  three (`1`, `777`, and `2024`).

For seed 2024, the baseline emitted one warning during the run, but the final
sender-side scores contained no agent at or below `-3`; the isolated trace
therefore passed with `0 warnings issued, 0 needed`. This is why the precise
claim is about the validator's **final expected-warning set**, not that no
agent ever crossed the threshold.

The strongest robustness result is that three isolated runs received zero
testimony yet both validators were green. This does not make either validator
incorrect relative to its implemented property; it shows that those
properties do not establish end-to-end evidence delivery or liveness.

## What I learned

1. A deterministic trace makes a partition a strong paired experiment: the
   emitted workload can be held fixed while delivery changes.
2. Validator evidence boundaries matter. A `send` record establishes intent
   to communicate, not receipt or effect.
3. `reputation_scoring` is a per-agent coverage check, not a per-cheat
   accountability check.
4. `reputation_warnings` can reveal a deaf observer when sender-side scores
   imply warnings, but a clean result can also mean there was nothing in its
   final expected set.
5. A passing validator is evidence for the encoded invariant, not proof that
   the overall reputation mechanism functioned.

## What I would build next

I would add a delivery-aware liveness validator. It would compare `send`,
`receive`, and `dropped` records for accountability messages and fail when a
coordinating agent receives zero inputs, or when an entire emitted message
class has zero delivery. For reputation specifically, it should distinguish:

- a report emitted by a trader;
- a report delivered to the observer;
- a score update caused by that report; and
- any warning required by the resulting score.

That would complement the current safety-style checks without changing their
documented meaning, and would make a disconnected observer visible even when
the expected-warning set is empty.

## Reproduce

From the repository root, with `nest-core[plugins]` installed:

```bash
nest run reputation \
  -o examples/reputation-observer-isolation/traces/reputation-baseline.jsonl

nest run examples/reputation-observer-isolation/reputation-observer-isolated.yaml \
  -o examples/reputation-observer-isolation/traces/reputation-isolated.jsonl

nest inspect examples/reputation-observer-isolation/traces/reputation-baseline.jsonl
nest inspect examples/reputation-observer-isolation/traces/reputation-isolated.jsonl
```

Run the validators:

```bash
python -c "
from pathlib import Path
from nest_core.validators import validate_trace

for label in ('baseline', 'isolated'):
    path = Path(
        f'examples/reputation-observer-isolation/traces/reputation-{label}.jsonl'
    )
    print(f'=== {label} ===')
    for result in validate_trace(path, 'reputation'):
        status = 'PASS' if result.passed else 'FAIL'
        print(status, result.name, '-', result.detail)
"
```

The built-in scenario is deterministic: rerunning a condition with the same
seed reproduces its trace.

To reproduce the supplementary paired-seed check:

```bash
mkdir -p examples/reputation-observer-isolation/traces/seed-sweep

for seed in 1 2 3 7 11 42 99 123 777 2024; do
  nest run reputation --seed "$seed" \
    -o "examples/reputation-observer-isolation/traces/seed-sweep/base-$seed.jsonl"
  nest run examples/reputation-observer-isolation/reputation-observer-isolated.yaml \
    --seed "$seed" \
    -o "examples/reputation-observer-isolation/traces/seed-sweep/iso-$seed.jsonl"
done
```

## Tools and help

I used the NANDA Town quickstart, scenario guide, CLI, JSONL traces, validator
source, Python's standard library, Git, and GitHub. Claude helped with
experiment planning, pressure-testing the hypothesis, and drafting early summaries.
Codex helped review the writing and structure of this README.
I chose the experiment, ran the scenarios, reviewed the traces, and verified the reported outputs.
No human help was used.
