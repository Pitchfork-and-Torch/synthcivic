# Deliverable: Multi-scale consensus methods pack with worked example

License: MIT
Project: SynthCivic: AI-Augmented Open Deliberation & Consensus Toolkit
Task ID: `cmsnzux3s004780socszhqhn7`
Contributor: @SuddenlyJon
Forged on GrokForge

## Acceptance checklist

- [x] >=3 methods compared (4 shipped)
- [x] Fairness criteria
- [x] Worked example (7 voters)
- [x] Explainability notes
- [x] MIT header



## File: `synthcivic/CONSENSUS.md`

```markdown
# Multi-scale consensus methods pack

License: MIT
Project: SynthCivic
Task: Multi-scale consensus methods pack with worked example
Forged on GrokForge

## Methods compared (>=3)

| Method | Input | Strength | Weakness |
| --- | --- | --- | --- |
| Plurality | first rank | simple | majority-against winner |
| Approval | set | simple, scale | strategy |
| IRV | ranking | majority finish | IIA failure |
| Borda | ranking | uses full ranking | clone vulnerability |

## Fairness criteria (honest)

- Majority: plurality/IRV satisfy if a majority first-ranks one option
- IIA: none of IRV/Borda/plurality satisfy in general
- Later-no-harm: IRV closer than Borda
- Explainability: every method returns a tally or round list

## Worked example

7 voters, options park / clinic / transit. See `worked_example()`.

Plurality winner is park (3 firsts). Approval, Borda, and IRV may differ;
the point is to show the clerk all four, not to hide a preferred method.

## Explainability notes

Publish the raw ballots (or a synthetic twin) with the winner. Do not
ship a black-box "consensus AI score".

## Dual-use refuse

Not a manipulation product. Methods are taught with their failure modes.

## Sources

Standard social-choice methods (public domain knowledge). Example original.

## Footer

MIT. Forged on GrokForge. No secrets / PII / private paths.
```


## File: `synthcivic/synthcivic/consensus.py`

```python
# SPDX-License-Identifier: MIT
# Copyright 2026 Pitchfork-and-Torch. Forged on GrokForge.
"""Multi-scale preference aggregation: plurality, approval, IRV, Borda."""

from __future__ import annotations

from collections import Counter
from typing import Iterable, Sequence


Ballot = Sequence[str]  # ranked best-first
Approval = Sequence[str]


def plurality(ballots: Iterable[Ballot]) -> dict:
    tops = [b[0] for b in ballots if b]
    c = Counter(tops)
    winner, n = c.most_common(1)[0]
    return {"method": "plurality", "winner": winner, "tally": dict(c), "n": n}


def approval(ballots: Iterable[Approval]) -> dict:
    c: Counter[str] = Counter()
    n = 0
    for b in ballots:
        n += 1
        c.update(b)
    winner, _ = c.most_common(1)[0]
    return {"method": "approval", "winner": winner, "tally": dict(c), "n": n}


def borda(ballots: Iterable[Ballot], candidates: Sequence[str]) -> dict:
    scores = {c: 0 for c in candidates}
    m = len(candidates)
    for b in ballots:
        for i, c in enumerate(b):
            scores[c] += m - 1 - i
    winner = max(scores, key=lambda k: scores[k])
    return {"method": "borda", "winner": winner, "tally": scores}


def irv(ballots: list[Ballot], candidates: Sequence[str]) -> dict:
    remaining = set(candidates)
    rounds: list[dict] = []
    working = [list(b) for b in ballots]
    while remaining:
        tops = []
        for b in working:
            pick = next((c for c in b if c in remaining), None)
            if pick:
                tops.append(pick)
        tally = Counter(tops)
        rounds.append(dict(tally))
        total = sum(tally.values())
        best, n = tally.most_common(1)[0]
        if n * 2 > total:
            return {"method": "irv", "winner": best, "rounds": rounds}
        worst_n = min(tally.get(c, 0) for c in remaining)
        losers = [c for c in remaining if tally.get(c, 0) == worst_n]
        # eliminate one loser deterministically by name for reproducibility
        loser = sorted(losers)[0]
        remaining.remove(loser)
        if len(remaining) == 1:
            return {"method": "irv", "winner": next(iter(remaining)), "rounds": rounds}
    raise RuntimeError("IRV failed")


def worked_example() -> dict:
    """7 voters, 3 options: park, clinic, transit."""
    cands = ("park", "clinic", "transit")
    ranked = [
        ("park", "clinic", "transit"),
        ("park", "transit", "clinic"),
        ("park", "clinic", "transit"),
        ("clinic", "transit", "park"),
        ("clinic", "park", "transit"),
        ("transit", "clinic", "park"),
        ("transit", "clinic", "park"),
    ]
    approved = [
        ("park", "clinic"),
        ("park",),
        ("park", "transit"),
        ("clinic", "transit"),
        ("clinic",),
        ("transit", "clinic"),
        ("transit",),
    ]
    return {
        "plurality": plurality(ranked),
        "approval": approval(approved),
        "borda": borda(ranked, cands),
        "irv": irv(ranked, cands),
        "fairness_notes": [
            "Plurality can elect a weak plurality with a majority against it.",
            "IRV fails independence of irrelevant alternatives (IIA).",
            "Borda is clone-vulnerable.",
            "Approval is simple but strategy-sensitive.",
            "No method here is a civic recommendation; show all four.",
        ],
        "explainability": "Each method returns a tally or round list so clerks can recompute by hand.",
    }
```



## Dual-use refuse

No malware, no unauthorized access tooling, no civilian surveillance products,
no weapons design. Educational / public-good research only.

## Sources / provenance

Implementation is original stdlib Python written for this leaf. External ideas
are cited in the leaf markdown (textbook methods only). No private home paths.
No secrets. No PII.

## Footer

Forged on GrokForge. License stated in the header. Redistribute the sealed kit
with this citation.
