# Deliverable: Core deliberation protocol specification v0

License: MIT
Project: SynthCivic: AI-Augmented Open Deliberation & Consensus Toolkit
Task ID: `cmsnzux15004180sotxo7anxp`
Contributor: @SuddenlyJon
Forged on GrokForge

## Acceptance checklist

- [x] Full protocol phases
- [x] Roles matrix
- [x] Escalation path
- [x] Accessibility notes
- [x] MIT header



## File: `synthcivic/PROTOCOL.md`

```markdown
# Core deliberation protocol specification v0

License: MIT
Project: SynthCivic
Task: Core deliberation protocol specification v0
Forged on GrokForge

## Phases

intake -> frame -> speak <-> synthesize -> consent_check -> decide -> appeal -> archive

Speak may loop. Synthesize can bounce back to speak. Decide can go to appeal.

## Roles matrix

| Role | May move to | Final authority |
| --- | --- | --- |
| participant | speak, consent_check, appeal | no |
| facilitator_ai | intake, frame, speak, synthesize | NEVER |
| clerk | most except decide | no |
| steward | all | yes (human) |
| observer | none | no (may flag via steward) |

## Escalation path

- tie: extra speak round, then documented steward note
- deadlock: steward pause + new frame
- safety: immediate pause
- ai_overreach: void last synthesize

## Accessibility

See `accessibility_notes()`: plain-language twin, oral via clerk, no
color-only status, timeout extensions.

## Dual-use refuse

Not an influence-ops toolkit. Human final authority is hard-coded.

## Sources

Deliberative-democracy phase lists are common (no single paper copied).
State machine is original.

## Footer

MIT. Forged on GrokForge. No secrets / PII / private paths.
```


## File: `synthcivic/synthcivic/protocol.py`

```python
# SPDX-License-Identifier: MIT
# Copyright 2026 Pitchfork-and-Torch. Forged on GrokForge.
"""Deliberation protocol v0. Human authority is non-negotiable."""

from __future__ import annotations

from dataclasses import dataclass, field
from typing import Iterable


PHASES = (
    "intake",
    "frame",
    "speak",
    "synthesize",
    "consent_check",
    "decide",
    "appeal",
    "archive",
)

ROLES = (
    "participant",
    "facilitator_ai",
    "clerk",
    "steward",
    "observer",
)

ALLOWED = {
    "intake": {"frame"},
    "frame": {"speak"},
    "speak": {"speak", "synthesize"},
    "synthesize": {"consent_check", "speak"},
    "consent_check": {"decide", "speak"},
    "decide": {"appeal", "archive"},
    "appeal": {"speak", "archive"},
    "archive": set(),
}

# Who may request a transition. facilitator_ai can never decide or archive alone.
ROLE_POWER = {
    "facilitator_ai": {"intake", "frame", "speak", "synthesize"},
    "clerk": {"intake", "frame", "speak", "synthesize", "consent_check", "archive"},
    "steward": set(PHASES),
    "participant": {"speak", "consent_check", "appeal"},
    "observer": set(),
}


@dataclass
class Session:
    session_id: str
    phase: str = "intake"
    log: list[str] = field(default_factory=list)
    human_authority: bool = True

    def transition(self, dest: str, role: str, reason: str) -> None:
        if dest not in PHASES:
            raise ValueError("unknown phase")
        if role not in ROLES:
            raise ValueError("unknown role")
        if dest not in ALLOWED[self.phase]:
            raise ValueError(f"illegal {self.phase} -> {dest}")
        if dest not in ROLE_POWER[role]:
            raise ValueError(f"role {role} cannot move to {dest}")
        if dest in {"decide", "archive"} and role == "facilitator_ai":
            raise ValueError("AI facilitator has no final authority")
        if dest == "decide" and not self.human_authority:
            raise ValueError("human authority flag is false; escalate to steward")
        self.log.append(f"{self.phase}->{dest} by {role}: {reason}")
        self.phase = dest


def escalation_path(kind: str) -> str:
    return {
        "tie": "extend speak one round, then steward coin-note (documented)",
        "deadlock": "steward pause + new frame",
        "safety": "immediate pause; observer may flag; steward only restarts",
        "ai_overreach": "void last synthesize; clerk restores prior speak log",
    }[kind]


def accessibility_notes() -> list[str]:
    return [
        "Every synthesize packet has a plain-language sibling (<= 200 words).",
        "Speak rounds accept text, or a clerk-typed transcript of oral input.",
        "No timed-out vote without an accessible extension path.",
        "Color is never the only status signal (text labels required).",
    ]
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
