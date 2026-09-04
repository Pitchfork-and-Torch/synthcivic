# Deliverable: Public ledger + audit schema for AI interventions

License: MIT
Project: SynthCivic: AI-Augmented Open Deliberation & Consensus Toolkit
Task ID: `cmsnzux5d004b80sot01k622q`
Contributor: @SuddenlyJon
Forged on GrokForge

## Acceptance checklist

- [x] Valid ledger schema
- [x] AUDIT.md
- [x] Example entries
- [x] Privacy balance
- [x] MIT header



## File: `synthcivic/AUDIT.md`

```markdown
# AUDIT.md  (public ledger + AI interventions)

License: MIT
Project: SynthCivic
Task: Public ledger + audit schema for AI interventions
Forged on GrokForge

## Ledger schema

See `schemas/ledger.schema.json`.

Each entry: seq, prev_hash, kind, actor_role, human_authority, redacted,
payload, ts_unix, hash = SHA-256(canonical JSON without hash).

Kinds: ai_intervention | decision | override | pause | note

## Privacy balance

- Default `redacted=true`
- Payload holds summaries, not raw speech
- No emails, phones, real names
- AI interventions must use actor_role=facilitator_ai
- AI cannot write `decision` entries (enforced)

## Example entries

`example_chain()` writes note -> ai_intervention -> override -> decision.

## Audit checklist

- [ ] Chain verifies (`Ledger.verify()`)
- [ ] Every AI act has a following human override-or-accept
- [ ] Decisions name a human role
- [ ] No PII keys present

## Dual-use refuse

Ledger is for civic transparency, not for tracking private persons.

## Sources

Hash chaining is textbook (git / blockchain 101). Schema original.

## Footer

MIT. Forged on GrokForge. No secrets / PII / private paths.
```


## File: `synthcivic/schemas/ledger.schema.json`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://grokforge.app/schemas/synthcivic/ledger.v0.json",
  "title": "SynthCivic ledger entry",
  "type": "object",
  "required": ["seq", "prev_hash", "kind", "actor_role", "human_authority", "redacted", "payload", "hash"],
  "properties": {
    "seq": { "type": "integer", "minimum": 0 },
    "prev_hash": { "type": "string", "minLength": 64, "maxLength": 64 },
    "kind": { "enum": ["ai_intervention", "decision", "override", "pause", "note"] },
    "actor_role": { "type": "string" },
    "human_authority": { "type": "boolean" },
    "redacted": { "type": "boolean" },
    "payload": { "type": "object" },
    "ts_unix": { "type": "integer" },
    "hash": { "type": "string", "minLength": 64, "maxLength": 64 }
  },
  "additionalProperties": false
}
```


## File: `synthcivic/synthcivic/ledger.py`

```python
# SPDX-License-Identifier: MIT
# Copyright 2026 Pitchfork-and-Torch. Forged on GrokForge.
"""Hash-chained public ledger for decisions and AI interventions."""

from __future__ import annotations

from dataclasses import dataclass, field
from typing import Any, Literal
import hashlib
import json
import time

Kind = Literal["ai_intervention", "decision", "override", "pause", "note"]

GENESIS = "0" * 64


def _canon(obj: dict[str, Any]) -> bytes:
    return json.dumps(obj, sort_keys=True, separators=(",", ":")).encode("utf-8")


@dataclass
class Ledger:
    entries: list[dict[str, Any]] = field(default_factory=list)

    def _prev(self) -> str:
        return self.entries[-1]["hash"] if self.entries else GENESIS

    def append(
        self,
        kind: Kind,
        actor_role: str,
        payload: dict[str, Any],
        *,
        redacted: bool = True,
        human_authority: bool = True,
    ) -> dict[str, Any]:
        if kind == "ai_intervention" and actor_role != "facilitator_ai":
            raise ValueError("ai_intervention must be logged as facilitator_ai")
        if kind == "decision" and actor_role == "facilitator_ai":
            raise ValueError("AI cannot author a decision entry")
        body = {
            "seq": len(self.entries),
            "prev_hash": self._prev(),
            "kind": kind,
            "actor_role": actor_role,
            "human_authority": human_authority,
            "redacted": redacted,
            "payload": payload,
            "ts_unix": int(time.time()),
        }
        body["hash"] = hashlib.sha256(_canon(body)).hexdigest()
        self.entries.append(body)
        return body

    def verify(self) -> bool:
        prev = GENESIS
        for i, e in enumerate(self.entries):
            if e["seq"] != i or e["prev_hash"] != prev:
                return False
            check = {k: v for k, v in e.items() if k != "hash"}
            if hashlib.sha256(_canon(check)).hexdigest() != e["hash"]:
                return False
            prev = e["hash"]
        return True


def example_chain() -> list[dict[str, Any]]:
    led = Ledger()
    led.append("note", "clerk", {"text": "session opened; 7 participants"})
    led.append(
        "ai_intervention",
        "facilitator_ai",
        {"action": "synthesize", "summary": "three clusters: park, clinic, transit"},
    )
    led.append("override", "steward", {"void": "none", "reason": "checked; ok"})
    led.append("decision", "steward", {"choice": "clinic", "method": "irv"})
    return led.entries
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
