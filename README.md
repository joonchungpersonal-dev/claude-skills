# Claude Code Skills

Multi-agent verification, code review, context management, and decision-analysis skills for [Claude Code](https://docs.anthropic.com/en/docs/claude-code).

## Skills

### veracity-555

Parallel fact-checking system. Decomposes any document into atomic claims (SAFE-style), then launches 3 waves of 5 verification agents per run — each wave attacks from a different angle. Supports iterative multi-run auditing with convergence detection.

- **16 agents per run** (1 decomposer + 15 verifiers across 3 waves)
- **6-point veracity scale** adapted from PolitiFact
- **Adversarial debate** (Tool-MAD) for disputed claims
- **Inter-run review protocol** with user approval before applying fixes
- **Self-audited**: scored 62/100 on its own SKILL.md, then 96.5/100 after iterative convergence (see Background)

### veracity-v2-experimental

Enhanced v2 of veracity-555 with 12 gap fixes. Adds a decomposition quality gate, citation verification, blind re-derivation, logical composition checking, confidence calibration, and smarter weighted consensus.

- **20 agents per run** across 5 waves (decomposition + quality gate + 3 verification waves)
- Experimental — use veracity-555 for production audits

### grill

Adversarial 2-phase code review. A Comprehensive Reviewer agent finds issues across 7 categories, then a GPTLens Critic agent validates each finding to eliminate false positives. Based on sources recommended by Boris Cherny (creator of Claude Code).

- **5C audit findings** (Condition, Criteria, Cause, Effect, Recommendation — IIA Standards 2410/2420)
- **OWASP risk scoring** (Likelihood x Impact, 0-9 scale)
- **GPTLens Auditor/Critic pattern** (Hu et al., IEEE TPS 2023)
- Sources: [trailofbits/claude-code-config](https://github.com/trailofbits/claude-code-config), [obra/superpowers](https://github.com/obra/superpowers)

### mental-models

Apply Charlie Munger's latticework of 35 mental models to any decision or problem. Structured multi-disciplinary analysis that surfaces blind spots, biases, and second-order effects.

- **35 models** across 7 disciplines (psychology, economics, biology, physics, engineering, mathematics, systems)
- **Full or brief mode** (`--brief` for key-questions-only)
- Source: Swedish Investor (Parts 1-6), Poor Charlie's Almanack

### context-engineer

Predicts context window overflow before it happens. Analyzes planned workflows, identifies chokepoints, and designs disk-as-memory architectures so complex multi-step tasks can run without crashing or losing information.

- **Information taxonomy**: 5 categories (IMPERATIVE, FACTUAL, REASONING, EPHEMERAL, RELATIONAL) with different fidelity requirements
- **Chokepoint prediction**: Identifies where workflows will exceed context limits
- **State file design**: Structured JSON that enables `/clear` and resume without degradation
- **7 anti-patterns** to flag (Fat Subagent Returns, Append Forever, etc.)
- **Victory condition**: If you can `/clear`, read your state file, and continue — your architecture is correct

### readme-audit

Audits a README against its actual project state. Built on the veracity-555 architecture but specialized for repository drift detection — every claim is verified against files on disk, package manifests, git history, and live URLs.

- **17 agents per run** (1 decomposer + 1 structural scanner + 15 verifiers)
- **12 claim categories**: VERSION, INSTALL, USAGE, FEATURE, DEPENDENCY, ARCHITECTURE, BADGE, CONFIG, LINK, TEMPORAL, LICENSE, CONTRIBUTOR
- **Health Score** (0-100), **Staleness Index**, **Completeness Score**
- Overall assessment: CURRENT / DRIFTING / STALE / UNRELIABLE

## Installation

Copy any skill directory into your Claude Code skills folder:

```bash
# Clone the repo
git clone https://github.com/joonchungpersonal-dev/claude-skills.git

# Copy the skill(s) you want
cp -r claude-skills/skills/veracity-555 ~/.claude/skills/
cp -r claude-skills/skills/context-engineer ~/.claude/skills/
cp -r claude-skills/skills/readme-audit ~/.claude/skills/
cp -r claude-skills/skills/grill ~/.claude/skills/
cp -r claude-skills/skills/mental-models ~/.claude/skills/
cp -r claude-skills/skills/veracity-v2-experimental ~/.claude/skills/
```

Then invoke in Claude Code:
```
/veracity-555 ~/path/to/document.md runs=1
/context-engineer
/readme-audit ~/path/to/repo
/grill
/mental-models "Should I accept this job offer?" --brief
```

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI
- Claude API access (Opus recommended for verification quality; Sonnet works for simpler documents)
- Token budget awareness: veracity-555 consumes ~800K-1M tokens per run; a 3-run audit uses 2.4M-3M tokens

## How It Works

The two verification skills (veracity-555 and readme-audit) are built on the same core architecture:

1. **SAFE Decomposition** (arXiv:2403.18802): Break documents into atomic, independently verifiable claims
2. **Parallel Verification Waves**: Multiple specialized agents check claims simultaneously, each from a different angle
3. **Supermajority Consensus**: 75% agent agreement threshold; disputed claims go through adversarial debate (Tool-MAD, arXiv:2601.04742)
4. **Iterative Convergence**: Multi-run mode applies fixes, re-audits, and can converge toward a stable score (convergence detection requires runs >= 4)

The context-engineer skill addresses the practical problem that these multi-agent workflows can exceed Claude's 200K context window. It provides a framework for externalizing state to disk so workflows can run indefinitely without information loss.

## Skills Pairing: Convergence Loop

The veracity-555 and context-engineer skills are designed to work together. For multi-run audits (`runs >= 2`), veracity-555 automatically applies context-engineer patterns:

- **Pre-flight analysis**: Estimates token budget and identifies chokepoints before the first run
- **State file**: Writes a `workflow-state.json` file (schema: `context-engineer/workflow-state/v1`) that tracks scores, findings, relationships, and decisions across runs
- **Checkpoint protocol**: Between runs, externalizes FACTUAL, REASONING, and RELATIONAL information to disk
- **Crash recovery**: If a session dies mid-audit, a new session reads the state file and continues from the next run

See [`workflows/convergence-loop.md`](workflows/convergence-loop.md) for a detailed walkthrough using the veracity-555 self-audit (74 → 96.5 over 6 runs, zero compactions) as a worked example.

## Background

These skills were developed during research on multi-agent fact verification. The veracity-555 skill was self-audited in two campaigns: an initial audit (Feb 2026) scored 62 → 80 → 84 over 3 runs, followed by a convergence loop (Mar 2026) that went 74 → 85 → 91 → 95.7 → 96.1 → 96.5 over 6 runs with 35 fixes. Key findings:

- **Attribution inflation** was the dominant error pattern — the skill systematically used slightly stronger language than primary sources warranted
- **Regression detection**: Fixes in Run 1 introduced 2 new factual errors caught in Run 2 (new content → new claims → new verification targets)
- **Convergence behavior**: Score deltas followed a predictable decay (null → +11 → +6 → +4.7 → +0.4 → +0.4)

The context-engineer skill was born from a practical failure: a veracity audit run consumed 3.67 MB of context (917K tokens), exceeding the 200K context window and crashing the session.

## Community Audit

We ran veracity checks across 247 skills from 4 popular repos. Full results in [`external-audits/results/`](external-audits/results/):

| Repo | Skills | Claims | Accuracy |
|------|--------|--------|----------|
| anthropics/skills | 17 | 54 | 94.4% |
| trailofbits/skills | 60 | 165 | 93.3% |
| obra/superpowers | 14 | 28 | 92.9% |
| K-Dense-AI/claude-scientific-skills | 156 | 285 | 88.4% |
| **Total** | **247** | **532** | **90.8%** |

See [`consolidated-report.md`](external-audits/results/consolidated-report.md) for the full report.

## License

MIT
