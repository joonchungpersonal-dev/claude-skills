# I built a veracity-checking skill for Claude Code. Running it on itself was humbling — here's what I learned.

I'm a researcher who uses Claude Code daily (sleep science background, University of Miami). As a scientist, factual accuracy isn't optional — citing the wrong paper or using the wrong number can undermine a study. So I built a Claude Code skill called `/veracity-tweaked-555` — in collaboration with Claude Code (Opus 4.6) — to systematically fact-check documents: it decomposes text into atomic claims and verifies each one via web search, using 16 parallel agents across 4 waves (1 decomposition + 3 verification). This entire project — the skills, the audits, the context engineering, this post — was a human-AI collaboration. Claude drafted the code and helped iterate on the implementation; I designed the methodology, made editorial decisions, and verified accuracy by running the veracity tool on everything, including this post.

## The self-audit: 62/100

The first thing I did was run it on its own SKILL.md.

It scored **62 out of 100**.

The skill I built to catch hallucinations had hallucinated facts in its own documentation. It had fabricated a performance statistic ("3x more accurate" for SAFE, which the paper never claims), inflated a paper's improvement claim ("+35.5%" was actually +5.5% over SOTA), and fabricated an acronym expansion for a real technique. These were claims that read well, sounded right, and were wrong.

After initial fixes it reached 80, then 84 after a third run. A week later, I ran a more rigorous convergence loop — 6 runs, 19 agents, 35 additional fixes — and it stabilized at 96.5/100. But the path wasn't a smooth climb. The fresh v3 audit actually *dropped* to 74 — because the v1 fixes had introduced new errors (an understated token cost and an incomplete tool list). Fixing things created new things to fix.

```
Veracity Score Over Time (9 audit runs)
100 ┤                                                   ●──── ● 96.5
 95 ┤                                             ●  95.7
 90 ┤                                       ● 91
 85 ┤                  ● 84           ● 85        ▲ v3 convergence
 80 ┤        ● 80        ╲                         (6 runs, 35 fixes)
 75 ┤                      ╲ ● 74
 70 ┤                       ╲╱  ← regression!
 65 ┤  ● 62                     (v1 fixes introduced
 60 ┤  ▲ v1 self-audit           new errors)
     └───────────────────────────────────────────────────────
      Feb 22                 Mar 1
```

That journey taught me more about LLM reliability than any paper I've read. The errors aren't random. They follow patterns: attribution inflation (slightly stronger language than the source warrants), plausible-but-fabricated identifiers (PMIDs, arXiv IDs that look real but point to different papers), and stale statistics presented as current.

## The context engineering problem

A single audit run generates ~917K tokens across 16 agents. Claude Code's 200K context window can't hold it. But the real problem isn't size — it's what gets lost when the window compresses.

When Claude Code compacts your conversation to stay within limits, it performs lossy compression. Regular facts survive reasonably well — names, numbers, function signatures. But **relational information** is the first casualty: causal chains between findings, cross-references, which fix caused which regression. "Finding A contradicts Finding B, which was caused by Fix C from Run 2" — that kind of reasoning gets crushed.

This is categorically different from losing ephemeral details like timestamps or file paths. Relational context is what makes multi-step reasoning *work*. Without it, you get an agent that can recall individual facts but can't reason about how they connect.

I solved this by building a companion skill called `/context-engineer` that predicts overflow before it happens and externalizes relational state to JSON files on disk. The design test: if you can `/clear` your entire conversation and resume from the state file alone, the architecture is correct. If you can't, you're still depending on context that will eventually be compressed away.

In my experience, after a few compactions the agent loses track of how findings relate to each other — which fix caused which regression, which claim contradicts which other claim. Individual facts survive reasonably well, but the connections between them don't. Externalizing state to disk bypasses this entirely: the JSON file on disk doesn't get compacted.

## Running it on my own skills

Encouraged by the self-audit, I ran veracity checks on all 6 of my Claude Code skills before releasing them publicly. The results:

- My `/grill` skill — which I built by consolidating patterns from GPTLens (Hu et al.), Trail of Bits' security config, obra/superpowers, and IIA/OWASP standards into a 2-agent code review architecture — had a **fabricated paper title** in its attribution section. The citation crediting the GPTLens source paper looked perfect (authors, venue) but the title was fabricated and the year was wrong. The actual paper has a completely different name. The irony: the section designed to give proper credit contained a hallucinated citation.
- The same skill misattributed the 5C audit framework to COSO (a different standards body) instead of IIA Standards 2410/2420. This error appeared in multiple locations across the file.
- My `/context-engineer` skill had internal inconsistencies — the prose said "5-10K tokens" while a table in the same file said "5-15K tokens" for the same metric.

12 total fixes across 6 skills. All passed at 95+ on 3 consecutive runs after corrections.

The takeaway: **I am not immune to this, and I don't think anyone is.** If you're writing Claude Code skills, you almost certainly have claims in your SKILL.md files that you'd want to correct. Not because you were careless, but because LLMs generate plausible-sounding facts that pass human review.

## Trying it on community skills

After auditing my own work, I wanted to see if the tool could be useful to the broader community. I ran veracity checks across 247 skills from 4 popular repos — not to find fault, but to understand the landscape and see if the patterns I found in my own work showed up elsewhere.

The short answer: **the ecosystem is in great shape.** Here's what I found across 532 verified claims:

### The good news (and there's a lot of it)

- **Zero fabricated papers** across all 4 repos. Every citation I checked — Jumper 2021, Cialdini 2021, Kocher 1996, Fagan 1976 — was real, with correct authors and venues.
- **Zero invented libraries or tools.** Every package, API, and tool referenced actually exists.
- **Zero malicious content.** No prompt injection, no hidden instructions.
- **Security guidance is overwhelmingly accurate.** All CWE numbers, vulnerability patterns, and cryptographic standards verified across Trail of Bits' 60 skills.
- **Academic citations are well-sourced.** obra/superpowers cites Cialdini (2021) and Meincke et al. (2025) with correct stats, affiliations, and publishers.
- **Overall accuracy: 91%** of verifiable claims checked out.

| Repo | Skills | Claims | Accuracy |
|------|--------|--------|----------|
| anthropics/skills | 17 | 54 | 94.4% |
| trailofbits/skills | 60 | 165 | 93.3% |
| obra/superpowers | 14 | 28 | 92.9% |
| K-Dense-AI/claude-scientific-skills | 156 | 285 | 88.4% |
| **Total** | **247** | **532** | **90.8%** |

### The main pattern: staleness

37% of all errors were **stale database statistics** — not fabrication, just entropy. Databases update; skill files don't. Ensembl "250 species" is now 348 vertebrate species (4,800+ eukaryotic genomes), ZINC "230M compounds" is now 5.9 billion, and so on. A `last_verified` field in skill frontmatter could catch this before users hit it.

## Limitations (being honest about what this can and can't do)

1. **Single-pass verification** — the community audit used one agent per claim via web search. My self-audits used 16 agents with multi-run convergence. The community results are therefore less thorough.
2. **Web search recency bias** — some "errors" may have been correct when the skill was written. I tried to note this where applicable.
3. **Procedural claims untested** — I verified factual assertions (numbers, citations, API endpoints), not whether the workflows produce correct results.
4. **LLM-as-judge** — the auditor agents are themselves LLMs, subject to the same hallucination risks. I mitigated the highest-impact findings by confirming them empirically via NCBI, PyPI, NVD, and HuggingFace APIs.
5. **Meta-circularity** — a tool built by an LLM, auditing content written by LLMs, checked by LLM agents. I can't fully escape this. The self-audit (62 → 96.5) and empirical API confirmations are my best defense.
6. **No runtime testing** — I checked registries and APIs for the top findings but didn't execute every code example in every skill.

## Where the real risk lives: newly created skills

One important caveat about these results: the repos I audited are **well-established**. anthropics/skills and obra/superpowers have tens of thousands of stars, active maintainers, and the benefit of many eyes over time. Trail of Bits brings professional security expertise. K-Dense-AI has domain scientists reviewing the content. These repos have had the benefit of community feedback, issue reports, and iterative refinement.

The skills that worry me more are the ones that *haven't* had that scrutiny — skills freshly generated by an LLM in a single session, dropped into `~/.claude/skills/`, and never audited. That's exactly what happened with my own skills before I started self-auditing.

I think running a veracity check after creating a new skill should be standard practice, the same way you'd run tests after writing code. It takes a few minutes and catches the kinds of errors that are invisible to the person who just wrote the file.

This is especially relevant if you're working solo or outside a large engineering team. If you're at a company with code review culture, someone might catch a wrong citation or a stale API reference. But a lot of us — researchers, indie developers, solo practitioners — don't have that safety net. We're building with Claude Code in domains where we're the only expert in the room, and there's no second pair of eyes on the SKILL.md. Automated veracity checking fills that gap. It's not a replacement for peer review, but it's a meaningful substitute when peer review isn't available.

## What I'm sharing and why

Both skills (`/veracity-tweaked-555` and `/context-engineer`) are open source at [joonchungpersonal-dev/claude-skills](https://github.com/joonchungpersonal-dev/claude-skills), along with the full audit results (JSON files + consolidated report) from the community skills audit.

I'm sharing these because the most valuable thing I learned is that **self-auditing catches errors that survive human review** — not because you're careless, but because LLM-generated errors are plausible by nature. They read well, sound right, and are sometimes wrong.

I have no illusions that this is the most efficient implementation — I'm a scientist, not a software engineer. I'm confident there are people in this community who could make the verification faster, the token usage lower, the false positive rate better, and the convergence tighter. I welcome improvements, forks, and PRs. The methodology matters more to me than the implementation.

Happy to answer questions about the methodology, the context engineering approach, or any specific finding. I'm here to contribute.

---

**Repo**: [joonchungpersonal-dev/claude-skills](https://github.com/joonchungpersonal-dev/claude-skills)
**Full audit report**: `external-audits/results/consolidated-report.md`
**Raw data**: `external-audits/results/*.json` (all 532 claims with verdicts and evidence)

---

## Appendix: Veracity audit of this post

I ran the veracity process on this post before publishing. Here's what I found and corrected:

**Errors caught and fixed before posting:**
- *"I had read that citation dozens of times"* — exaggeration. I had reviewed the attribution section but "dozens of times" overstates it. Replaced with an accurate description of why the error survived review.
- *"Six convergence runs and 35 fixes later, it stabilized at 96.5/100"* — conflated two separate audit sessions. The initial self-audit (Feb 22) went from 62 → 80 → 84 over 3 runs. A separate convergence loop a week later (Mar 1) went from 74 → 96.5 over 6 runs with 35 fixes. Corrected to accurately describe both phases.
- *"/grill presented as my own skill"* — misleading by omission. `/grill` is a composite skill I built by consolidating patterns from GPTLens (Hu et al.), Trail of Bits' security config, obra/superpowers, IIA/OWASP standards, and Fagan inspection limits. My contribution was the 2-agent architecture, panel-tagging, dashboard logging, paper trail archiving, and the Think & Verify step. Corrected to properly credit sources.
- *"invented a technique name"* — imprecise. The technique (Tool-MAD) is real. What was fabricated was the acronym expansion. Corrected.

**Claims verified as accurate:**
- 62/100 initial score, 80/100 after v1 fixes, 74/100 regression at v3 start, 96.5/100 after v3 convergence (confirmed via audit trail logs)
- 917K tokens per audit run (confirmed via MEMORY.md session logs), 200K context window (confirmed via context-engineer SKILL.md)
- 12 total fixes across 6 skills (confirmed via workflow-state.json)
- All table numbers (532 claims, 49 errors, per-repo breakdowns) verified against aggregated JSON results
- GPTLens fabricated title, COSO misattribution in multiple locations, context-engineer internal inconsistency — all confirmed via git history
- Ensembl 348 species via REST API, PMID verification via NCBI E-utils, caracal via PyPI — all confirmed empirically

**Additional fixes during manual review:**
- *Skill name wrong* — post said `/veracity-555` throughout. Actual name is `/veracity-tweaked-555`. Fixed in all occurrences.
- *Score graph showed smooth climb* — the graph went 62 → 74 → 84 → 85 → ... which hid the actual 80 → 74 regression between v1 and v3. The v1 fixes introduced new errors (understated token cost, incomplete tool list) that the fresh v3 audit caught. Corrected graph to show the real trajectory: 62 → 80 → 74 → 85 → 91 → 95.7 → 96.1 → 96.5.
- *"84/100 after initial fixes"* — actual v1-Run2 score was 80/100. Corrected.
- *Fidelity degradation chart (90% → 75% → 55% → 30%)* — presented with specific percentages that were never rigorously measured. Removed the chart and replaced with honest prose about observed behavior.
- *No mention of human-AI collaboration* — the entire project was built with Claude Code. Added explicit note about the collaboration and division of labor.
- *"Encouraged (and humbled)"* — removed "and humbled" as editorializing.
- *Two sentences overstating personal review* — deleted "The 62/100 starting score wasn't from a hastily written file..." and the GPTLens anecdote. Both were more dramatic than accurate.
- *"Self-auditing works"* — vague. Replaced with "self-auditing catches errors that survive human review."

**Automated veracity Run 1 (score: 87.2%, 116 facts, 5 MIXED):**
- *Chart said "8 audit runs"* — actually 9. The Feb 22 series had 3 runs (62 → 80 → 84), not 2. Corrected chart and prose.
- *Appendix said "62 → 84"* — skipped the intermediate 80. Corrected to "62 → 80 → 84."
- *Ensembl "250 species is now 4,800+ genomes"* — conflated species count with genome count. Corrected: "348 vertebrate species (4,800+ eukaryotic genomes)."
- *"917K tokens confirmed via context-engineer SKILL.md"* — the 917K figure is from MEMORY.md session logs, not context-engineer SKILL.md. Corrected source attribution.
- *GPTLens citation "correct authors, year, venue"* — the year was also wrong (2024 vs 2023). Removed "year" from list of correct elements.

**Automated veracity Run 2 (score: 90.7%, 124 facts, 0 MIXED):**
- All 5 Run 1 fixes confirmed applied correctly. 0 regressions.
- *"4 verification waves"* — Wave 0 is decomposition, not verification. Changed to "4 waves (1 decomposition + 3 verification)."

**Automated veracity Run 3 (score: 90.2%, 128 facts, 0 MIXED/FALSE):**
- All prior fixes confirmed intact. 0 regressions across 3 runs.
- All 15 numbers in the community audit table independently verified against raw JSON data.
- *"5 separate locations"* for COSO misattribution — audit trail evidence shows 4 confirmed locations. Changed to "multiple locations."
- Score is structurally capped at ~91-92% by 11 MOSTLY TRUE facts that cannot be upgraded with available evidence (single-measurement claims like "917K tokens," heuristics like "relational info is the first casualty," and counting methodology differences).

**Convergence accepted at Run 3 (90.2%). 0 FALSE, 0 MOSTLY FALSE, 0 MIXED across 128 facts. Remaining MOSTLY TRUE facts are honest approximations (single measurements, heuristics, soft counts) that cannot be upgraded with available evidence.**

**Remaining caveats:**
- *"All CWE numbers... verified across Trail of Bits' 60 skills"* — slight overclaim. We verified the CWE numbers, vulnerability patterns, and standards that our auditor agents specifically checked, not literally every CWE reference in every file. Softened to "overwhelmingly accurate."
- *"91% accuracy"* — this is the "not found to be wrong" rate: (532 - 49) / 532 = 90.8%. If you exclude the 51 unverifiable claims, the confirmed-correct rate is 432/481 = 89.8%. Both framings are defensible; I used the more generous one.
- *"Both skills are open source"* — true at time of posting, contingent on the repo being public (it is).
