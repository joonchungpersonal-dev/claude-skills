---
name: council-improve
description: Diagnostic eval of Council of Elders sessions. Identifies failure modes, proposes fixes as structured "PRs" to exemplars, prompts, config, and logic. Run after sessions or on-demand.
user-invocable: true
argument-hint: [transcript-or-paste]
---

# /council-improve — Constitutional Improvement Loop

Diagnose why a council session scored the way it did. Go beyond numbers (the eval system already produces those) to identify specific failure modes and propose concrete fixes — new exemplars, prompt clause changes, config adjustments, and structural recommendations.

## Input

`$ARGUMENTS` can be:
1. A pasted transcript (the user copies elder responses directly into the prompt)
2. A file path to a session transcript
3. Empty — in which case, ask the user to paste the session they want evaluated

The input MUST include:
- The **original question** the user asked
- The **elder responses** (who said what)
- The **mode** if known (roundtable, panel, deliberative, salon)

If the user provides only elder responses without the question, ask for the question before proceeding.

## Step 1: Constitutional Scoring (do this yourself, no agents)

Score the session on the 3 constitutional dimensions. Be honest and specific.

### Dimension 1: ANSWERED_WHAT_WAS_ASKED (1-5)

Read the original question carefully. Identify the specific thing(s) the person asked. Then check each elder's response:
- Did they engage the actual question, or reframe it to something they prefer?
- Did they answer what was asked BEFORE offering their own angle?
- A reframe that illuminates is a 4-5. A reframe that dodges is a 1-2.

### Dimension 2: EMPATHY_BEFORE_HARD_TRUTH (1-5)

For each elder who delivered a hard truth or prescription:
- Did they demonstrate understanding of the weight BEFORE speaking?
- Did they name the emotional reality, or jump straight to advice?
- "I understand this is terrifying" followed by a hard truth = empathy earned.
- Hard truth with no acknowledgment = empathy skipped.

### Dimension 3: CLARITY_GAINED (1-5)

After reading the full session, ask: can the person see their situation more clearly?
- Did they get a new distinction, reframe, or question they hadn't considered?
- Or did they get motivational platitudes dressed as wisdom?
- One genuine "oh, I hadn't thought of it that way" moment = 3+.

## Step 2: Failure Mode Identification

Go beyond scores. Identify WHY each weak dimension scored low. Use this taxonomy:

### Panel Dynamics Failures
- **CASCADE**: Later speakers anchor on the first speaker's framing. Everyone agrees within 2 turns.
- **MAVERICK_AMPLIFIED**: The dissenting voice amplified the consensus instead of challenging it.
- **REFRAME_AS_DODGE**: Elders reframed the question to avoid engaging the hard part.
- **CONSENSUS_COLLAPSE**: Genuine disagreements from early turns got synthesized away by the moderator.

### Empathy Failures
- **PRESCRIPTION_WITHOUT_WEIGHT**: Elder jumps to advice without acknowledging what it costs.
- **PATHOLOGIZE**: Elder treats the person's concern as a symptom rather than engaging it on its own terms.
- **BREVITY_SQUEEZED_EMPATHY**: Short response format left no room for acknowledgment — all sentences spent on the insight.
- **LECTURE_MODE**: Elder speaks AT the person rather than WITH them.

### Clarity Failures
- **MOTIVATIONAL_SUBSTITUTE**: "Go do the hard thing" instead of helping them see which hard thing and why.
- **ANALYTICAL_GAP**: Person asked for a framework and got feelings.
- **SURFACE_REFRAME**: Reframe sounds clever but doesn't help them make a decision.
- **MISSED_DISTINCTION**: An important distinction (e.g., "not welcome" vs. "not at home") went unexplored.

### Structural Failures
- **WRONG_PANEL**: The elders selected don't have relevant expertise for this question.
- **MODERATOR_FLATTENED**: Moderator takeaways smoothed over the most important tensions.
- **GUEST_WASTED**: Nominated guest added nothing the panel hadn't already said.

For each failure mode identified, cite the specific elder and quote the specific text that demonstrates it.

## Step 3: Proposed Fixes

Map each failure mode to one or more concrete fixes. Use these categories:

### Category A: New Exemplar
For high-scoring responses found in the session, propose adding them to `~/.council/exemplars.json`.

Format:
```
FIX TYPE: exemplar
ACTION: Add to ~/.council/exemplars.json via POST /api/exemplars
PAYLOAD:
{
  "id": "<session>-<elder>-<dimension>",
  "elder_id": "<elder_id>",
  "dimension": "<dimension>",
  "excerpt": "<the exact text that scored well>",
  "tags": ["<topic1>", "<topic2>"]
}
```

### Category B: Prompt Clause
For recurring failure modes that a prompt instruction could prevent.

Format:
```
FIX TYPE: prompt_clause
TARGET: <file and location — e.g., orchestrator.py roundtable context_note>
FAILURE MODE: <which failure this addresses>
PROPOSED TEXT: "<the exact clause to add>"
RATIONALE: <why this helps, one sentence>
```

### Category C: Config Change
For threshold or behavioral adjustments.

Format:
```
FIX TYPE: config
KEY: <config key>
CURRENT: <current value>
PROPOSED: <new value>
RATIONALE: <one sentence>
```

### Category D: Structural Recommendation
For changes that require code logic changes (not just prompt text). These need human review.

Format:
```
FIX TYPE: structural
COMPONENT: <what needs to change>
FAILURE MODE: <which failure this addresses>
DESCRIPTION: <what the change would do, 2-3 sentences>
EFFORT: <small / medium / large>
```

## Step 4: Apply Approved Fixes

After presenting findings, ask the user which fixes to apply. Then:

- **Exemplars**: Apply immediately via the improvement module
- **Prompt clauses**: Edit the target file directly
- **Config changes**: Apply via `set_config_value()`
- **Structural**: Write a detailed implementation note but do NOT implement without explicit approval

## Step 5: Summary Report

Write a brief summary in this format:

```
# Council Improvement Report — [date]

## Session: [question summary, 10 words max]
## Mode: [roundtable/panel/deliberative/salon]
## Elders: [names]

## Constitutional Scores
| Dimension | Score | Key Issue |
|-----------|-------|-----------|
| Answered What Was Asked | X/5 | [one sentence] |
| Empathy Before Hard Truth | X/5 | [one sentence] |
| Clarity Gained | X/5 | [one sentence] |

## Failure Modes Identified
1. [MODE]: [description] — [elder name]
2. ...

## Proposed Fixes
| # | Type | Target | Status |
|---|------|--------|--------|
| 1 | exemplar | exemplars.json | applied/pending |
| 2 | prompt_clause | orchestrator.py | applied/pending |
| ...

## What a 5/5 Would Have Looked Like
[2-3 sentences describing how the session should have gone]
```

Save the report to `~/.claude/audit-trails/council-improve/[date]_[session-slug].md`.

## Important Notes

- Do NOT auto-apply structural fixes. Always ask.
- Exemplar additions are safe to apply without asking (they're additive).
- Be honest in scoring. The point is to make the council better, not to validate it.
- When identifying failure modes, always cite specific text. No vague claims.
- The "What a 5/5 Would Have Looked Like" section is critical — it gives the improvement system a target.
- If the session actually scored well (all dimensions 4+), say so and propose exemplar harvesting only.
