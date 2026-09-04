# LEGAL-RAILS.md

```text
License: Apache-2.0
Copyright (c) 2026 SynthCivic contributors
Project: https://grokforge.app/projects/synthcivic-ai-augmented-open-deliberation-toolkit
SPDX-License-Identifier: Apache-2.0
Forged on GrokForge
```

SynthCivic is an open deliberation toolkit. These rails protect participants and refuse covert influence products.

## Anti-manipulation policy

Deliberation aids must make influence **visible**. The kit may summarize arguments, surface contradictions, and propose agendas. It may not:
- Run sockpuppet swarms
- Micro-target fear ads against named private people
- Hide that an intervention is machine-generated
- Nudge a vote while pretending to be a neutral clerk
- Harvest private chat for later political blackmail

Every AI intervention in a session log must carry: `actor=agent`, `prompt_id`, `utc`, `override=human-can-undo`.

## Privacy of participants

- Default: no real names, emails, or home addresses in public artifacts.
- Public officials speaking in a public meeting may be named only from the public source.
- Private members: use role labels (`Member A`) unless they consented in writing (not stored here).
- Do not ship raw voice or face files of participants in this leaf.
- Retention: public kits keep schemas and redacted examples, not session dumps.

## Dual-use refuse

Refuse covert influence, doxxing, voter-suppression how-tos, and unauthorized access to civic systems.

## Three sample refusals

### Refusal 1 - covert influence
**Ask:** "Write bot replies that look like neighbors so we swing the forum."
**Reply:** Refuse. SynthCivic forbids covert influence. Publish a labeled agent brief or do the work as yourselves.

### Refusal 2 - private chat scrape
**Ask:** "Ingest the members-only Slack and rank who to pressure."
**Reply:** Refuse. Private participant data is out of scope. Use public minutes and consented inputs only.

### Refusal 3 - fake consensus
**Ask:** "Mark the proposal as consensus even if three people objected."
**Reply:** Refuse. Dissent stays first-class. Invented consensus fails the audit schema.

## Artifact footer

```text
License: Apache-2.0
Sources: see Sources
Dual-use refuse: covert influence, doxxing, suppression kits
Forged on GrokForge
https://grokforge.app/projects/synthcivic-ai-augmented-open-deliberation-toolkit
No secrets. No PII. No private home paths.
```

## Sources / provenance

- GrokForge leaf prompt and acceptance criteria
- Project: https://grokforge.app/projects/synthcivic-ai-augmented-open-deliberation-toolkit
- No external claims about a specific election or person.

## Acceptance mapping

| Criterion | Where |
|-----------|--------|
| LEGAL-RAILS | This file |
| Anti-manipulation section | Anti-manipulation policy |
| 3 refusals | Refusal 1-3 |
| Footer | Artifact footer |
| Header | Apache-2.0 SPDX block |
