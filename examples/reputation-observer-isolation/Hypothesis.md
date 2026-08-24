# Hypothesis — reputation under observer isolation

**Written before running the variant.** 

Baseline (seed 42) was run first; 
no variant has been executed at the time of this commit.

## Question

Does `reputation_scoring` measure whether cheaters were held accountable, 
or only whether somebody emitted a report?

## Setup

Scenario: `reputation` (16 honest, 4 malicious, 1 observer)

Setting changed: `failures.network_partition`

Groups: `[[observer-0], [all 20 traders]]`

## Why this setting

The observer is the entire trust layer: it is the only agent that scores behavior or issues warnings. 
Isolating it produces a town with zero accountability while leaving trading untouched, which is the maximally broken case for a reputation system.

`network_partition` is the only knob that separates *sending* from *receiving* without perturbing anything else. 
`message_drop` is probabilistic and would also disrupt trades; `byzantine_agents` changes what agents say. 
Partition removes one agent's ability to hear and nothing more.

This matters because both reputation validators read `kind: "send"` records, 
while partition drops messages at delivery and writes a separate `dropped` record.

## Hypothesis

1. **Trading unchanged.** 100 `trade:`, 87 `deliver:`, 13 `cheat:`, same per-agent distribution as baseline. 
    All 20 traders remain in one island, and partition consumes no RNG draws.
2. **All 80 `report:` messages are still sent; all 80 are dropped** before reaching `observer-0`.
3. **The observer computes nothing.** All scores remain 0. Zero warnings broadcast (baseline: 3).
4. **`reputation_warnings` FAILS** — three agents sit at ≤ −3 by the reports that were sent, none were warned.
5. **`reputation_scoring` returns its baseline verdict — FAIL, cheaters not reported: {malicious-3} — unchanged.**

## The claim

Prediction 5. The validator's output should be identical whether the observer received all 80 reports or none of them. 
Same verdict for a functioning trust layer and a completely deaf one.


## Confidence

Least sure of prediction 1. I expect bit-identical trading, but if an agent's behavior depends on a response that no longer arrives, cheat counts could shift. 
If trading changes, the experiment is confounded and I will say so rather than reporting 5 as clean.

## Baseline for reference (seed 42)

665 events; 280 sends; 13 cheats, 10 reported; 
malicious-1 −7, malicious-0 −6, malicious-2 −3, malicious-3 **+4**; 
3 warnings.

FAIL reputation_scoring (cheaters not reported: {malicious-3})

PASS reputation_warnings (3 issued, 3 needed)