# /council-qa — Post-Discussion QA Analysis

Analyze the latest Council of Elders discussion capture for UX bugs, compliance issues, and visual problems.

## When to Use

Run `/council-qa` after completing a discussion in the Council of Elders app. The app automatically captures HTML and PDF snapshots of every completed discussion to `~/.council/qa/`.

## Workflow

1. Read `~/.council/qa/manifest.json` to find the latest capture entry
2. Read the `.html` file for text-level analysis (roster, sentence counts, content quality)
3. Read the `.pdf` file for visual/layout analysis (if available)
4. Run all checks from the rubric below
5. Output a structured bug report

## Rubric

### P0 — Roster Integrity (Critical)
- **Only selected elders spoke**: Compare `metadata.elder_ids` against elder names that actually appear in the HTML. Flag any elder who spoke but was NOT in the selected roster (maverick intrusion bug).
- **All selected elders spoke**: Every elder in `metadata.elder_ids` should have at least one response in the transcript. Flag any missing elders.
- **Moderator identity**: The moderator should NOT be attributed as a specific elder. Moderator text should appear in moderator-styled elements, not elder cards.

### P1 — Sentence Compliance
- **Count sentences per elder**: For each elder response, count the number of sentences (split on `.!?` followed by space or end). Compare against the configured `metadata.response_length`:
  - `brief` = 1-2 sentences (hard cap: 2)
  - `moderate` = 3-4 sentences (hard cap: 4)
  - `detailed` = 5-8 sentences (hard cap: 9)
  - `extended` = 10-15 sentences (hard cap: 16)
  - `unlimited` = no cap (skip sentence compliance check)
- Flag any elder whose response significantly exceeds the expected range (>2x the upper bound is a hard fail).

### P1 — Response Quality
- **On-topic**: Each elder response should relate to `metadata.question`. Flag responses that are entirely off-topic.
- **In-character**: Elder responses should reflect their known expertise and era. Flag obvious anachronisms or misattributed knowledge.
- **Not truncated**: Responses should end with complete sentences. Flag responses that end mid-word or mid-sentence.
- **No duplicate content**: Flag if two elders give nearly identical responses (>80% text overlap).

### P2 — UI Completeness
- **Eval badge**: If constitutional evaluation is enabled, check for the presence of an eval badge element (`.eval-badge` or similar) or a pending eval poll.
- **Action buttons**: Check for podcast download button, transcript export button, and journal save prompt.
- **Follow-up prompt**: The system message "Ask a follow-up question..." should appear after the discussion ends.

### P2 — Format Integrity
- **No broken HTML**: Check for unclosed tags, malformed entities, or raw HTML visible as text.
- **Proper elder attribution**: Each elder response should have a visible name, title, and era in the header.
- **Moderator formatting**: Moderator messages should use the moderator card style, not elder card style.

### P3 — Visual Layout (PDF only)
- **No overlapping elements**: Check that text and UI elements don't visually overlap.
- **Readable text**: All text should be legible (not clipped, not white-on-white).
- **Complete capture**: The PDF should contain the full discussion, not just the visible viewport.

## Output Format

```markdown
## Council QA Report — {timestamp}

**Mode**: {mode} | **Elders**: {count} | **Question**: "{question truncated to 80 chars}"

### Issues Found

#### P0 — Critical
- [ ] {description} — **{category}**

#### P1 — High
- [ ] {description} — **{category}**

#### P2 — Medium
- [ ] {description} — **{category}**

#### P3 — Low
- [ ] {description} — **{category}**

### Pass Summary
- Roster Integrity: PASS/FAIL
- Sentence Compliance: PASS/FAIL ({details})
- Response Quality: PASS/FAIL
- UI Completeness: PASS/FAIL
- Format Integrity: PASS/FAIL
- Visual Layout: PASS/FAIL/SKIPPED (no PDF)

### Suggested Fixes
1. {fix description} — affects `{file_path}`
```

## Arguments

- No arguments: analyze the most recent capture
- `--all`: analyze all captures in the manifest
- `--since YYYY-MM-DD`: analyze captures since the given date
- `--capture N`: analyze the Nth most recent capture (1 = latest)

## Notes

- HTML analysis is always available; PDF analysis requires Electron (not browser-only mode)
- The skill reads files only — it never modifies the app or triggers discussions
- Captures are fire-and-forget from the app side; missing captures indicate a bug in the capture pipeline itself
