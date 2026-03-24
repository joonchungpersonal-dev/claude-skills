# /ux-committee — Agent Committee UX Testing

Launch a multi-agent UX testing committee that evaluates any web or desktop app from the perspective of real, diverse users. Produces a consolidated visual report with screenshots, timing data, and prioritized fix recommendations.

## When to use

Before any public release, after significant UI changes, or when the user says "test the UX," "test like a real user," "visual test," or "ux-committee."

## Architecture

The committee has two layers:

### Layer 1: Specialist Agents (5 parallel)

Each agent runs Playwright against the app (browser or Electron) and owns one domain:

**S1: Visual Inspector**
- Takes screenshots at every meaningful state transition
- Checks layout integrity (no overflow, no overlapping elements)
- Tests responsive design (desktop → tablet → mobile viewports)
- Verifies theme switching (dark/light) doesn't break anything
- Checks icon/image loading, empty states, error states

**S2: Timing Auditor**
- Measures time-to-first-paint from cold start
- Measures time-to-first-streaming-token after submitting a question
- Measures time-to-response-complete for each mode
- Flags anything >3s for first paint, >5s for first token, >60s for completion
- Tests perceived performance: are loading indicators shown? Progress feedback?

**S3: Interaction Tester**
- Walks through every clickable element systematically
- Tests keyboard navigation (Tab order, Enter/Escape, shortcuts)
- Tests input validation (empty submit, very long input, special characters)
- Tests all settings toggles and dropdowns
- Tests navigation flows (settings → back, profile panel → close, new chat)

**S4: Error Recovery Tester**
- Submits questions while disconnected (kills Flask mid-test)
- Tests the retry button after errors
- Tests behavior with invalid/expired API keys
- Sends adversarial inputs (prompt injection, XSS attempts)
- Tests what happens when streaming is interrupted (click Stop, navigate away)

**S5: Accessibility & Polish Checker**
- Checks color contrast ratios (WCAG AA minimum)
- Verifies all interactive elements have hover/focus states
- Checks for orphaned tooltips, modals, overlays
- Verifies scroll behavior during streaming
- Checks print/export styling

### Layer 2: User Persona Pool (8 parallel agents)

Each agent embodies a specific user type and interacts with the app naturally, reporting friction points:

**P1: "Impatient Tech Founder"**
- Clicks fast, expects instant responses, uses keyboard shortcuts
- Will bail if anything takes >5 seconds without feedback
- Tries to do 3 things at once
- Judges the app in the first 10 seconds

**P2: "Non-Technical Parent"**
- Doesn't know what Ollama, API keys, or LLMs are
- Types slowly, reads everything on screen
- Gets confused by jargon
- Needs clear guidance at every step

**P3: "Academic Researcher"**
- Asks very long, detailed, multi-part questions
- Expects substantive depth, not platitudes
- Wants to export/save/cite responses
- Compares answers across elders for consistency

**P4: "Gen Z Student"**
- Uses slang, emoji, one-word questions
- Low patience, swipes away fast
- Tests on mobile viewport
- Questions about identity, relationships, career anxiety

**P5: "Skeptical Hacker News Commenter"**
- Tries to break things immediately
- Tests adversarial prompts, looks for prompt leaks
- Judges the tech stack, checks console for errors
- Will post "this is just a ChatGPT wrapper" if not impressed

**P6: "Power User / Enthusiast"**
- Customizes everything (response length, discussion mode, elder selection)
- Imports custom elders, generates podcasts, saves to journal
- Tests every feature path
- Wants keyboard shortcuts and efficiency

**P7: "Mobile-Only User"**
- 375px viewport, touch interactions only
- Limited data connection (test with throttling)
- Fat-finger taps, accidental scrolls
- Judges by mobile experience since they don't have a laptop open

**P8: "Grieving / Emotionally Vulnerable User"**
- Asks deeply personal questions about loss, meaning, crisis
- Needs warmth, not clinical responses
- Safety-critical: should never receive harmful advice
- Tests the emotional tone of responses

## Execution

### Step 1: Detect the target

Identify:
- Is there a running dev server? (check localhost ports)
- Is there a packaged app? (check for .app, .dmg, .exe in build output directories)
- What framework? (Electron, web-only, mobile)
- What are the key UI entry points?

### Step 2: Launch specialist committee (Layer 1)

Launch all 5 specialist agents in parallel using the Agent tool. Each agent:
1. Opens the app via Playwright (browser mode or Electron mode)
2. Runs their domain-specific test scenarios
3. Takes timestamped screenshots
4. Records metrics and findings
5. Returns a structured report

### Step 3: Launch user persona pool (Layer 2)

Launch all 8 persona agents in parallel. Each agent:
1. Opens a fresh app instance
2. Interacts as their persona would — naturally, with realistic pacing
3. Reports friction points, confusion, and delight moments
4. Rates the experience 1-10 from their persona's perspective
5. Lists their top 3 "I would leave because..." pain points

### Step 4: Consolidate

Merge all 13 agent reports into a single HTML report:
- **Executive summary**: Overall UX score (average of persona ratings), critical blockers, top 5 fixes
- **Screenshot gallery**: All screenshots from specialists, organized by flow
- **Timing dashboard**: All performance metrics with pass/fail thresholds
- **Persona feedback grid**: Each persona's experience, friction points, and rating
- **Prioritized fix list**: Issues ranked by (number of personas affected × severity)
- **Adversarial results**: What broke, what held

### Step 5: Present

Show the user the report and ask: "Want me to fix the top issues?"

## Output Files

```
tests/ux_committee/
├── report.html              # consolidated visual report
├── metrics.json              # all timing and scoring data
├── screenshots/              # all PNGs organized by agent
│   ├── S1_visual/
│   ├── S2_timing/
│   ├── S3_interaction/
│   ├── S4_error/
│   ├── S5_accessibility/
│   ├── P1_tech_founder/
│   ├── P2_parent/
│   └── ...
└── persona_reports/          # individual persona narratives
    ├── P1_tech_founder.md
    ├── P2_parent.md
    └── ...
```

## Prerequisites

- `pip install playwright && playwright install chromium`
- App must be running (dev server or packaged app)
- For Electron mode: packaged app must be built (`cd desktop && npm run make`)
