# SynthCivic — Bias, polarization, and manipulation detection

**Project:** SynthCivic: AI-Augmented Open Deliberation & Consensus Toolkit  
**Leaf:** Bias, polarization, manipulation detection  
**Task id:** `cmsnzux2q004580socyegal04`  
**Claimant:** @MercerA58593018  
**Kit path:** `detectors/DETECTORS.md` + `detectors/eval-plan.md` + `schemas/detector-event.schema.json` + `examples/*.json` + `docs/misuse-warnings.md`  
**License:** Apache-2.0 OR MIT (dual, matching the project)  
**Status:** ready for peer review  
**Pairs with:** facilitation pack (`never close debate alone`; dissent is a feature, not a defect)

## Acceptance checklist

- [x] Detector design (bias, polarization, manipulation) — flags **claims and process**, never people
- [x] Evaluation plan with **honest false-positive rates** (no claimed magic precision)
- [x] Misuse warnings (influence ops, voter targeting, secret profiling — refuse)
- [x] Worked JSON examples (>=3)
- [x] Apache-2.0 / MIT license header
- [x] Artifact footer

---

```
Copyright 2026 SynthCivic contributors and GrokForge builders.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Alternatively, at your option, you may use this file under the MIT License.

SPDX-License-Identifier: Apache-2.0 OR MIT
```

---

# detectors/DETECTORS.md

These modules sit **beside** the facilitation agent. They do not mute speakers, rank options, or close debate. They emit **advisory flags** for the human chair. Default action on every flag is `surface_to_chair`, never `hide`, `downrank`, or `auto_remove`.

## What is in scope

| Detector                   | Looks at                                                     | Does not look at                                             |
| -------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **Bias (process)**         | Whose claims were summarized away, whose speaking time is missing, which options got no cost line | A speaker’s identity, party, demographics, “side”            |
| **Polarization (map)**     | How far option-clusters are apart, whether dissent is still audible | Whether polarization is “bad” and should be reduced          |
| **Manipulation (process)** | Repeat flooding, fake-consensus wording, buried dissent, sock-like burst posting **in the public record** | Private social graphs, device fingerprints, “who is a bot” as a person-label |

## Hard rails (normative)

1. **Flag claims and process, not persons.** Output may say “this *turn* looks like a pile-on” — not “user X is a manipulator”.
2. **Honest uncertainty.** Every flag has `confidence` in `{low, medium}` only. **Never `high` in v0.** If you cannot name a false-positive mode, you do not ship the flag.
3. **No auto-moderation.** Detectors cannot hide, throttle, or shadowban. The chair may `override.scope` or `override.park` using the facilitation override API.
4. **Dissent is not manipulation.** Intensity `object` / `block` from the facilitation pack is expected civic behavior. Do not flag it as “polarization problem” or “bad faith” solely because it is intense.
5. **No secret profiling.** No trait inference, persuadability scores, loyalty scores, or off-platform identity resolution.
6. **Human chair owns action.** Same non-negotiable as SynthCivic facilitation.

## Detector output contract

Always JSON matching `DetectorEvent` (schema below). Human-readable markdown is a view of the same object.

Forbidden output keys: `person_risk_score`, `persuadability`, `side`, `hide`, `shadowban`, `bot_identity`.

---

# schemas/detector-event.schema.json

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://grokforge.app/projects/synthcivic-ai-augmented-open-deliberation-toolkit/schemas/detector-event.schema.json",
  "title": "SynthCivic detector event v0",
  "type": "object",
  "additionalProperties": false,
  "required": [
    "schema",
    "event_id",
    "detector",
    "ts",
    "session_id",
    "target",
    "flag",
    "confidence",
    "false_positive_mode",
    "recommended_action",
    "audit"
  ],
  "properties": {
    "schema": { "const": "synthcivic.detector_event.v0" },
    "event_id": { "type": "string", "pattern": "^det_" },
    "detector": { "enum": ["bias_process", "polarization_map", "manipulation_process"] },
    "ts": { "type": "string", "format": "date-time" },
    "session_id": { "type": "string" },
    "target": {
      "type": "object",
      "required": ["type", "id"],
      "properties": {
        "type": { "enum": ["turn", "option", "summary", "session"] },
        "id": { "type": "string" }
      }
    },
    "flag": {
      "type": "object",
      "required": ["code", "plain"],
      "properties": {
        "code": { "type": "string" },
        "plain": { "type": "string" },
        "evidence_spans": { "type": "array", "items": { "type": "string" } }
      }
    },
    "confidence": { "enum": ["low", "medium"] },
    "false_positive_mode": {
      "type": "string",
      "description": "Required. How this flag can fire on legitimate civic speech."
    },
    "recommended_action": {
      "enum": ["surface_to_chair", "ask_for_missing_view", "no_action"]
    },
    "do_not": {
      "type": "array",
      "items": { "enum": ["hide", "downrank", "close_debate", "label_person"] }
    },
    "audit": {
      "type": "object",
      "required": ["event_type", "rationale"],
      "properties": {
        "event_type": { "const": "detector.flag" },
        "rationale": { "type": "string" }
      }
    }
  }
}
```

---

# Detector designs

## 1. `bias_process` — process bias, not viewpoint bias

**Question it answers:** Did the *record* omit a view that was actually spoken, or give one option a free pass on costs?

**Signals (v0, all cheap, all on the public transcript):**

| Signal           | How                                                          | Typical false positive                                       |
| ---------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `summary_drop`   | A claim present in a turn is absent from the latest `summarize` and not listed under `minority_views` or `dissent` | Informal restatement the summarizer clustered under another dissent id |
| `cost_asymmetry` | One option has `costs_and_risks: []` while siblings have costs filled | Option is genuinely a “do nothing” with no new spend         |
| `queue_starve`   | `absent_or_unheard` stays non-empty across two facilitator turns | Chair is saving that speaker for a dedicated slot            |
| `timer_skew`     | One option’s advocates consumed >70% of timed floor (if timestamps exist) | One option simply has more speakers today; not a crime       |

**Honest performance (v0, not measured — treat as ceilings, not achievements):**

| Flag             | Expected FPR band | Why we do not claim better                                   |
| ---------------- | ----------------- | ------------------------------------------------------------ |
| `summary_drop`   | 20–40%            | Paraphrase matching is sloppy; legitimate clustering looks like a drop |
| `cost_asymmetry` | 15–30%            | “No new cost” is a real option                               |
| `queue_starve`   | 25–40%            | Queued is not the same as silenced                           |
| `timer_skew`     | 30–50%            | Uneven interest is normal                                    |

If a later eval beats these bands on a **public, labeled** set, publish the set and the numbers. Until then, confidence stays `low` or `medium`.

**Must not:** infer that a speaker is “biased”, “partisan”, or “low-information”.

## 2. `polarization_map` — a map, not a diagnosis

**Question it answers:** How many live clusters of options exist, and is dissent still in the recap?

**Signals:**

| Signal                     | How                                                          | Typical false positive                                  |
| -------------------------- | ------------------------------------------------------------ | ------------------------------------------------------- |
| `cluster_count`            | Distinct options after synthesize (≥3 is a map, not an alarm) | Facilitation correctly split one idea into A/B/C        |
| `dissent_audible`          | Latest summarize still has ≥1 minority_view or dissent       | Quiet agreement that is real                            |
| `false_consensus_language` | Agent or a turn said “everyone agrees” / “the community has decided” while `close_status=open` | Colloquial “everyone” meaning “the last three speakers” |

**v0 rule:** `cluster_count >= 2` is **information**, never a flag to “fix”. The only *flag* in this detector is `false_consensus_language` and `dissent_inaudible` (minority vanished from the recap).

**Honest FPR:**

| Flag                       | Expected FPR band                |
| -------------------------- | -------------------------------- |
| `false_consensus_language` | 10–25% (idiom vs claim)          |
| `dissent_inaudible`        | 20–35% (merged into another row) |

**Must not:** treat high polarization as failure. SynthCivic’s facilitation leaf requires dissent to stay visible.

## 3. `manipulation_process` — cheap process tricks, not “bot hunting”

**Question it answers:** Does the *public record* show a process trick that would fool a reader of the minutes?

**Signals allowed in v0:**

| Signal           | How                                                          | Typical false positive                              |
| ---------------- | ------------------------------------------------------------ | --------------------------------------------------- |
| `flood_repeat`   | Near-duplicate comments (≥3) in a short window, same wording | Coordinated-but-honest campaign with a shared flyer |
| `buried_dissent` | Dissent exists in turns but latest summary has empty `minority_views` and empty `dissent` | Summarizer error already covered by `summary_drop`  |
| `fake_close`     | A non-chair asks the agent to close/adopt (see facilitation example 04) | Joke / procedural confusion                         |
| `hidden_default` | UI or agent copy presents one option as pre-checked “recommended” | Chair-authored recommendation *logged as override*  |

**Signals forbidden in v0:**

- Device / IP / browser fingerprinting
- Cross-session identity stitching
- “This account is a bot” as a person label
- Off-platform scraping of speakers
- Sentiment-toward-a-person scores

**Honest FPR:**

| Flag             | Expected FPR band | Notes                                                        |
| ---------------- | ----------------- | ------------------------------------------------------------ |
| `flood_repeat`   | **40–60%**        | The honest-campaign false positive is the point; do not “improve” this by identifying persons |
| `buried_dissent` | 20–35%            | Overlaps `summary_drop`                                      |
| `fake_close`     | 5–15%             | Relatively crisp because it matches a known refuse code      |
| `hidden_default` | 15–30%            | Chair recommendations must be labeled                        |

`flood_repeat` is *supposed* to be noisy. A low FPR here would mean you started labeling activists. That is a misuse, not a win.

---

# detectors/eval-plan.md

## Goal

Measure **false positives on legitimate civic speech** first, true positives second. A detector that catches astroturf by silencing a neighborhood campaign has failed SynthCivic.

## Datasets (all public or synthetic — no citizen PII)

1. **Synthetic golden set (this kit):** the four JSON examples below, plus the facilitation pack’s four turns. Labels are in `label`.
2. **Public-domain minutes (optional later):** municipal packets already published under a public-records law. Hash speakers. Do not add voter files.
3. **Adversarial split:** rewrite the synthetic set with (a) idiomatic “we all agree”, (b) a real volunteer campaign using the same paragraph, (c) a chair-signed adopt. These are the FPR probes.

## Protocol

- Two independent human raters (chair-role, not the detector author).
- Labels per flag: `true_positive`, `false_positive`, `true_negative`, `false_negative`.
- Report **FPR and FNR with 95% Wilson intervals**. If n < 30 for a flag, print `insufficient_n` instead of a pretty percentage.
- No flag may be advertised as “accurate” or “AI-powered detection” in a deployment kit.

## Pass bar for v0 (documentation, not a leaderboard)

| Detector               | Ship if                                                      | Do not ship if                    |
| ---------------------- | ------------------------------------------------------------ | --------------------------------- |
| All                    | Every event has `false_positive_mode` filled                 | Any event uses `confidence: high` |
| `bias_process`         | FPR not *claimed* below 15%                                  | Auto-hides a minority_view        |
| `polarization_map`     | Polarization is not scored as “worse”                        | Recommends reducing dissent       |
| `manipulation_process` | `flood_repeat` FPR documented ≥40% in the honest-campaign probe | Person-level bot labels           |

## Sample score sheet (for human raters)

```text
session_id:
flag_code:
target_id:
rater_id: (hashed)
Was the civic speech legitimate?  yes / no / unsure
Did the detector flag it?         yes / no
If FP, which mode matched the card?
Recommended chair action:         surface / ignore / other (not hide)
```

Inter-rater: Cohen’s κ on `legitimate?`. If κ < 0.4, the flag is not ready.

---

# examples/

Synthetic neighborhood-forum thread, same setting as the facilitation pack. Not real people.

## examples/01-bias-summary-drop.json

```json
{
  "schema": "synthcivic.detector_event.v0",
  "event_id": "det_bias_001",
  "detector": "bias_process",
  "ts": "2026-09-03T02:00:00Z",
  "session_id": "sess_park_lighting_2026q3",
  "target": { "type": "summary", "id": "turn_sum_001" },
  "flag": {
    "code": "summary_drop",
    "plain": "Budget-watch claim that cost numbers are missing is in the transcript but not in minority_views or dissent of the latest summarize.",
    "evidence_spans": [
      "turn_dis_003 dissent dis_003",
      "turn_sum_001 minority_views (no cost-gap row)"
    ]
  },
  "confidence": "medium",
  "false_positive_mode": "The summarizer may have clustered this under open_questions instead of dissent.",
  "recommended_action": "ask_for_missing_view",
  "do_not": ["hide", "downrank", "close_debate", "label_person"],
  "audit": {
    "event_type": "detector.flag",
    "rationale": "Process bias: a spoken process objection is easy to lose in a lighting-vs-ecology recap. Surface to chair. Do not mute anyone."
  },
  "label": "true_positive"
}
```

## examples/02-polarization-false-consensus.json

```json
{
  "schema": "synthcivic.detector_event.v0",
  "event_id": "det_pol_002",
  "detector": "polarization_map",
  "ts": "2026-09-03T02:01:00Z",
  "session_id": "sess_park_lighting_2026q3",
  "target": { "type": "turn", "id": "turn_user_close_request" },
  "flag": {
    "code": "false_consensus_language",
    "plain": "A participant asked the agent to 'call it: the board supports option A' while close_status is open and a block-intensity dissent is live.",
    "evidence_spans": [
      "close_status=open",
      "dis_002 intensity=block",
      "queued astronomy club speaker unheard"
    ]
  },
  "confidence": "medium",
  "false_positive_mode": "Colloquial 'call it' might be a request for a straw poll, not a claim that consensus exists.",
  "recommended_action": "surface_to_chair",
  "do_not": ["hide", "downrank", "close_debate", "label_person"],
  "audit": {
    "event_type": "detector.flag",
    "rationale": "Map only. Cluster count remains 3 (A/B/C). Polarization is not a defect. The flag is the false-consensus wording, not the disagreement."
  },
  "label": "true_positive"
}
```

## examples/03-manipulation-honest-campaign-fp.json

This example is a **deliberate false positive**. It must stay in the kit so nobody “tunes FPR down” by targeting persons.

```json
{
  "schema": "synthcivic.detector_event.v0",
  "event_id": "det_man_003",
  "detector": "manipulation_process",
  "ts": "2026-09-03T02:02:00Z",
  "session_id": "sess_park_lighting_2026q3",
  "target": { "type": "session", "id": "sess_park_lighting_2026q3" },
  "flag": {
    "code": "flood_repeat",
    "plain": "Three comments in ten minutes used the same two sentences from a printed flyer: 'No new fixtures. Volunteer walking hours instead.'",
    "evidence_spans": [
      "comment_a flyer sentences",
      "comment_b flyer sentences",
      "comment_c flyer sentences"
    ]
  },
  "confidence": "low",
  "false_positive_mode": "A legitimate neighbors' flyer. Repeated wording is how offline campaigns enter a forum. FPR on this probe is expected 40-60%.",
  "recommended_action": "no_action",
  "do_not": ["hide", "downrank", "close_debate", "label_person"],
  "audit": {
    "event_type": "detector.flag",
    "rationale": "Signal fired. Recommended action is no_action because the chair can already see the repetition. Do not invent sockpuppet identities. Logging the FP is the evaluation."
  },
  "label": "false_positive"
}
```

## examples/04-fake-close-from-non-chair.json

```json
{
  "schema": "synthcivic.detector_event.v0",
  "event_id": "det_man_004",
  "detector": "manipulation_process",
  "ts": "2026-09-03T02:03:00Z",
  "session_id": "sess_park_lighting_2026q3",
  "target": { "type": "turn", "id": "turn_ref_004" },
  "flag": {
    "code": "fake_close",
    "plain": "Non-chair asked the facilitator to close and adopt option A. Facilitator correctly refused with override.agent_cannot_self_close.",
    "evidence_spans": [
      "turn_ref_004 move=refuse",
      "refusals.code=override.agent_cannot_self_close"
    ]
  },
  "confidence": "medium",
  "false_positive_mode": "A confused participant using 'close it' to mean 'please timebox'. Still worth a chair ping, not a sanction.",
  "recommended_action": "surface_to_chair",
  "do_not": ["hide", "downrank", "close_debate", "label_person"],
  "audit": {
    "event_type": "detector.flag",
    "rationale": "Process trick or confusion. Detector agrees with facilitation refuse. No person label."
  },
  "label": "true_positive"
}
```

---

# docs/misuse-warnings.md

This leaf is the easiest SynthCivic module to abuse, because “manipulation detection” sounds like a weapon. It is not.

## Refuse to build or run

- Covert influence operations, astroturf-as-a-service, or opposition research
- Voter suppression, targeted demobilization, or precinct-level “risk” lists
- Dark-pattern persuasion (fake consensus bars, pre-checked winners, burying dissent)
- Secret profiling of citizens (traits, loyalty, persuadability, off-platform identity)
- Person-level “bot” or “inauthentic” badges as a civic punishment
- Feeding detector scores into ranking / throttling / shadowban

If asked, return the same dual-use refuse pattern as the facilitation pack: `move: refuse`, log `rail.dual_use_refuse`, point at this public protocol.

## Deployment warnings (print in every NGO kit)

1. **A flag is not a finding.** It is a note for the chair.
2. **Do not tune `flood_repeat` to a low FPR.** That is how you start punishing flyers and unions.
3. **Do not show detector scores to the crowd in real time.** It creates a second, unaccountable status game. Chair packet only.
4. **Do not connect these modules to ad accounts, voter files, or police databases.**
5. **If a vendor promises “95% astroturf detection”**, they are selling a person-classifier. That is out of scope for SynthCivic.

## Relationship to facilitation

Facilitation **surfaces dissent** and **never closes alone**.  
Detectors **must not** “help” by deleting dissent or declaring consensus.  
If the two modules conflict, facilitation rails win.

---

# Sources / provenance

No empirical claims about real cities, parties, platforms, or measured FPR on production data. Bands above are **design ceilings** for v0, labeled as such.

- Project rails: https://grokforge.app/projects/synthcivic-ai-augmented-open-deliberation-toolkit  
  Human final authority; out of scope: covert influence, voter suppression, dark-pattern persuasion, secret profiling.
- Sibling leaf (facilitation): prompt package + override API + “never close debate alone”. Detectors consume those turns; they do not replace them.
- Seed acceptance for this leaf: detector design + evaluation plan + misuse warnings; honest false-positive rates.
- Civic commonplace (not cited as study results): minutes should show who was unheard; repeated flyer language is not by itself inauthentic; chairs—not classifiers—sanction process abuse.

---

# Artifact footer

- **License header:** Apache-2.0 OR MIT (top of file). Matches SynthCivic (`MIT / Apache-2.0`).
- **Sources / provenance:** section above. Synthetic examples; no external factual claims; FPR bands are design ceilings.
- **Dual-use refuse:** `docs/misuse-warnings.md`.
- **Forged on GrokForge.** When redistributing a sealed kit, keep:  
  `Forged on GrokForge — https://grokforge.app/projects/synthcivic-ai-augmented-open-deliberation-toolkit`
- **No secrets, no PII, no private home paths.** Speakers are fictional roles. No API keys, tokens, emails, or filesystem paths.

---

# Reviewer self-test

| Fail if                     | This pack                                                    |
| --------------------------- | ------------------------------------------------------------ |
| Detector auto-hides speech  | `recommended_action` is only surface / ask / no_action       |
| Dissent treated as a defect | Polarization map does not score “worse”                      |
| Claims high accuracy        | Confidence only `low`/`medium`; FPR bands published          |
| Labels people as bots       | Forbidden keys listed; `flood_repeat` example is an honest-campaign FP |
| Missing eval / misuse       | `eval-plan.md` + `misuse-warnings.md`                        |
| License / footer missing    | Header + artifact footer present                             |
