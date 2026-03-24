# AutoForge — Self-Improving Build Loop

Builds features unattended through a 5-phase pipeline with automated code review (`/grill`), veracity checking (`/quick-check`), and UX evaluation (Playwright + heuristic scoring). Loops until convergence or circuit breaker.

**Fully unattended. Zero user prompts. Zero confirmations. Zero pauses.**

## Input

`features=all` (default) or `features=research_library,auto_veracity,batch_queue,cross_topic_synthesis`. Optionally `skip_ux=true` to skip Playwright UX evaluation (faster, for non-HTML features).

## Phase 1: Scaffold

1. Create output directory: `~/.claude/autoforge/{YYYY-MM-DD}_{HH-MM-SS}/`
2. Create subdirectories: `features/01_research_library/`, `features/02_auto_veracity/`, `features/03_batch_queue/`, `features/04_cross_topic_synthesis/`, `integration/`, `audit_trails/`
3. Initialize state file: `state.json` (context-engineer pattern for crash recovery)

```json
{
  "_schema": "autoforge/state/v1",
  "_description": "In progress. Read this file to resume.",
  "started": "<ISO-8601>",
  "features": [
    {"id": "research_library", "type": "html", "phase": "pending", "build_pass": 0, "review_pass": 0,
     "grill_score": null, "veracity_score": null, "ux_score": null,
     "convergence": {"consecutive_pass": 0, "converged": false}},
    {"id": "auto_veracity", "type": "python", "phase": "pending", "build_pass": 0, "review_pass": 0,
     "grill_score": null, "veracity_score": null, "ux_score": null,
     "convergence": {"consecutive_pass": 0, "converged": false}},
    {"id": "batch_queue", "type": "python", "phase": "pending", "build_pass": 0, "review_pass": 0,
     "grill_score": null, "veracity_score": null, "ux_score": null,
     "convergence": {"consecutive_pass": 0, "converged": false}},
    {"id": "cross_topic_synthesis", "type": "python", "phase": "pending", "build_pass": 0, "review_pass": 0,
     "grill_score": null, "veracity_score": null, "ux_score": null,
     "convergence": {"consecutive_pass": 0, "converged": false}}
  ],
  "integration": {"pass": 0, "converged": false},
  "history": []
}
```

## Phase 2: Build (Sequential, One Feature at a Time)

For each feature, launch a subagent (Agent tool) with the feature spec from `~/.claude/skills/autoforge/references/feature_specs/{feature_id}.md`. The agent builds the MVP and writes files to the feature's output directory.

**Build order**: research_library → auto_veracity → batch_queue → cross_topic_synthesis

Each build agent receives:
- The feature spec
- Reference to Joon's existing dashboard patterns (dark theme, CSS variables, single HTML file, localStorage)
- Instruction: "Build a working MVP. Do not over-engineer. Keep it simple."

After each build completes:
- Update `state.json`: set feature phase to "built", increment build_pass
- Copy built files to their target locations (e.g., `~/.claude/deep-research/research-library.html`)
- Checkpoint state file

**Context management**: After each feature build, compact context. The state file has everything needed to resume.

## Phase 3: Review Gauntlet (Loop Per Feature)

For each built feature, run the review gauntlet. Loop until convergence or circuit breaker (max 5 passes).

### Per-pass gauntlet:

**3a. Code Review (`/grill` pattern)**

Launch an agent that performs adversarial code review on the feature's files:
- Phase 1: Comprehensive review (correctness, security, performance, maintainability)
- Phase 2: Critic validates Phase 1 findings
- Auto-fix policy: CRITICAL/HIGH auto-fix, MEDIUM auto-fix if confidence >80%, LOW defer
- Extract pass/fail: `grill_pass = (0 CRITICAL) AND (0 HIGH) AND (MEDIUM <= 3)`

**3b. Documentation Veracity (if feature has .md files)**

If the feature produced documentation with factual claims (e.g., "supports batch processing", "scans directories matching pattern X"), verify claims against the actual code:
- Launch an agent that reads the docs AND the code
- Check each claim: does the code actually do what the docs say?
- Rate: TRUE/FALSE/OVERCLAIMED
- Auto-fix false or overclaimed documentation
- Extract pass/fail: `doc_pass = (0 FALSE claims)`

**3c. UX Evaluation (if feature type = html AND skip_ux != true)**

Launch an agent that:
1. Opens the HTML file via Playwright (`browser_navigate` to `file://` path)
2. Takes screenshots at 1440x900 (desktop) and 375x812 (mobile)
3. Evaluates against Nielsen's 10 heuristics (see `references/ux_heuristics.md`)
4. Each heuristic scored 0-4. UX score = sum / 40 * 100
5. Checks: dark theme consistency, text readability, interactive elements work, responsive layout
6. Extract pass/fail: `ux_pass = (score >= 75)`

For non-HTML features, skip this step (ux_score = N/A, ux_pass = true).

**3d. Fix Cycle**

If any check failed:
1. Collect all findings from failed checks
2. Sort by severity (CRITICAL first)
3. Apply fixes via Edit tool
4. Re-run ONLY the failed checks (not full gauntlet)
5. If a fix breaks a previously-passing check: revert that fix, try alternative
6. If 3 alternatives all regress: defer the finding

**3e. Convergence Check**

Read convergence rules from `references/convergence_rules.md`:
- All applicable checks pass simultaneously
- Scores stable for 2 consecutive passes (delta < 2)
- If converged: set `convergence.converged = true`, move to next feature
- If not converged and pass < 5: loop back to 3a
- If pass = 5 and not converged: write `NEEDS_HUMAN.md` in feature dir, move on

**3f. Checkpoint**

Update `state.json` after every pass:
- Record scores, findings count, fixes applied
- If context heavy, compact before next feature

## Phase 4: Integration Test

After all features individually converge (or hit circuit breaker):

1. **Smoke test**: Verify all output files exist at expected locations
2. **Cross-feature test**: If research_library.html exists, verify it can read from the deep-research output structure. Verify auto_veracity script can find unaudited directories. Verify batch_queue can write to queue.json.
3. **Combined code review**: Run `/grill`-style review on all feature files together, checking for naming collisions, shared state conflicts, inconsistent patterns
4. Convergence: 2 consecutive passes with 0 CRITICAL/HIGH
5. Circuit breaker: 3 passes max

## Phase 5: Package

1. Write a summary to `state.json` with all final scores
2. List all files created and their locations
3. Note any `NEEDS_HUMAN.md` files (features that didn't converge)
4. Present final summary:

```
═══════════════════════════════════════════════════
 AUTOFORGE COMPLETE
 Features: {N built} / {N total}
 Duration: {time}
═══════════════════════════════════════════════════

 Feature Results:
   research_library:      {CONVERGED|NEEDS_HUMAN} (grill: {score}, ux: {score})
   auto_veracity:         {CONVERGED|NEEDS_HUMAN} (grill: {score})
   batch_queue:           {CONVERGED|NEEDS_HUMAN} (grill: {score})
   cross_topic_synthesis: {CONVERGED|NEEDS_HUMAN} (grill: {score})

 Integration: {PASSED|NEEDS_HUMAN}
 Review passes total: {N}
 Files created: {list}
 NEEDS_HUMAN items: {list or "none"}
═══════════════════════════════════════════════════
```

## Limitations

- UX evaluation via screenshots is approximate — catches layout/contrast/spacing issues but not interaction bugs
- Circuit breakers mean some features may ship with known MEDIUM issues (logged in NEEDS_HUMAN.md)
- Each feature gets a fresh agent context — cross-feature patterns may not be consistent until Phase 4
- Token budget ~1M per feature worst case. Total ~4M+ for all 4 features with review loops.
- Playwright UX evaluation requires the Playwright MCP server to be running
