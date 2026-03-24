# /council-ux-test — Council of Elders UX Integration Test

## Purpose
End-to-end test of the Council of Elders streaming pipeline. Validates event sequences, response quality, formatting, and visual consistency without launching the app.

## When to Use
- After modifying `orchestrator.py`, `app.py`, chat CSS, or chat JS
- Before pushing changes that affect the discussion flow
- When the user reports UX issues (cutoffs, formatting, visual bugs)

## Quick Start

### Frontend-only (no LLM, instant)
```bash
cd ~/council-of-elders && source venv/bin/activate && python scripts/test_ux.py --dry-run
```

### Full pipeline test (requires LLM API key)
```bash
cd ~/council-of-elders && source venv/bin/activate && python scripts/test_ux.py --mode all
```

### Single mode test
```bash
cd ~/council-of-elders && source venv/bin/activate && python scripts/test_ux.py --mode salon --response-length detailed
```

## What It Tests

### Backend Pipeline (per mode: panel, salon, roundtable, rap, poetry)
1. **Event sequence integrity** — every `elder_start` has matching `elder_done`, every `moderator_start` has `moderator_done`, stream ends with `discussion_done`
2. **Response lengths** — elder responses fall within the configured sentence/token limits for the response_length setting
3. **No orphaned events** — no chunks arrive without a preceding start event
4. **Interruption handling** — interrupted elders still get `elder_done` events
5. **Streaming** — responses arrive in multiple chunks (not dumped all at once)

### HTML Formatting
1. **No raw markdown** — `#`, `**`, `*` converted to proper HTML
2. **Balanced tags** — open/close tag counts match
3. **No control tag leakage** — `[DIRECT:]`, `[NOMINATE:]`, `[WRAP_UP]` stripped from output

### Frontend Static Analysis
1. **Elder color conflicts** — no elder colors too close to moderator green
2. **Moderator visual differentiation** — moderator card uses different layout than elder cards
3. **Event routing** — all stream handlers check for `__moderator__` routing

## Interpreting Results

```
[+] PANEL  (12.3s)  — PASS     # Green = good
[X] SALON  (15.1s)  — FAIL     # Red = has CRITICAL or HIGH issues
```

### Issue Severities
- **CRITICAL** — broken pipeline (backend error, no elders spoke, HTTP failure)
- **HIGH** — event sequence bugs (orphaned chunks, missing done events, routing gaps)
- **MEDIUM** — quality issues (short responses, unbalanced HTML, visual confusion)
- **LOW** — polish (raw markdown artifacts, low chunk count)

## After Running

1. If CRITICAL/HIGH issues found: fix the root cause, re-run the specific mode
2. If MEDIUM issues: assess whether they impact user experience
3. If all PASS: safe to push

## Options

| Flag | Description |
|------|-------------|
| `--mode MODE` | Test a single mode (panel/salon/roundtable/rap/poetry/all) |
| `--question "..."` | Custom test question |
| `--elders a,b,c` | Custom elder IDs (comma-separated) |
| `--response-length` | brief/moderate/detailed/extended |
| `--dry-run` | Frontend-only checks, no LLM calls |
| `--frontend-only` | Same as --dry-run |

## Files
- Test runner: `~/council-of-elders/scripts/test_ux.py`
- This skill: `~/.claude/skills/council-ux-test/SKILL.md`
