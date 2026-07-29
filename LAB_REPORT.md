# BugHound Lab Report

A record of every step performed in the lab and why it mattered.

## Part 1 — Explore the agentic workflow
**Steps:** Cloned the starter repo, set up the venv, ran the app in Streamlit,
read `bughound_app.py`, `bughound_agent.py`, and `reliability/risk_assessor.py`,
then ran `cleanish.py` in offline/Heuristic mode and read the Agent Trace.
**Purpose:** Understand BugHound as an agentic loop (PLAN → ANALYZE → ACT →
TEST → REFLECT) before changing anything, and identify where its behavior is
fragile. Observed the silent heuristic fallback when LLM output isn't parseable
JSON. Done offline to preserve the 20-request Gemini quota.

## Part 2 — Integrate an AI analyzer
**Steps:** Read the analyzer prompts, created `.env` with the Gemini API key,
traced the `analyze` method's choice between heuristic and LLM analysis, ran
Gemini mode on `cleanish.py` and `mixed_issues.py`, and made a reliability
change: `_normalize_issues` now validates `severity` against {Low, Medium, High}
and defaults anything else to Medium.
**Purpose:** Treat the model as a *tool* inside the system rather than the whole
solution, and stop malformed model output (bad severities) from flowing
unchecked into the risk layer. Gemini gave richer, context-aware issue detection
than the keyword heuristics.

## Part 3 — Propose fixes and evaluate risk
**Steps:** Read the `propose_fix` method and fixer prompts, ran Gemini mode on
`mixed_issues.py` and examined the diff, read the scoring rules in
`risk_assessor.py`, then tightened the auto-fix policy: `should_autofix` now
requires `score >= 90` **and** no Medium/High issues (previously just
`level == "low"`).
**Purpose:** Move from "assistant that comments" to "agent that acts safely."
Observed the LLM over-editing (deleting the print/TODO lines and narrowing the
except, which changes behavior). Closed the gap where a lone Medium issue
(score 80) would auto-apply without human review.

## Part 4 — Testing, reliability, and guardrails
**Steps:** Read and ran the existing tests (8 passing), noted how `MockClient`
forces the offline fallback path, then added
`test_single_medium_issue_does_not_autofix` — an offline test asserting the
tightened policy defers to a human. Confirmed it fails under the old policy and
passes under the new one. Full suite: 9 passed.
**Purpose:** Pressure-test the agent for failure modes (false positives,
over-editing, unusable output, unsafe confidence) and lock a guardrail's
*decision* in place with a repeatable test that costs no API quota.

## Part 5 — Reflection and model card
**Steps:** Completed all eight sections of `model_card.md`: system overview,
workflow, inputs/outputs, two reliability rules analyzed, two observed failure
modes, heuristic-vs-Gemini comparison, a human-in-the-loop trigger, and an
improvement idea (an over-editing guardrail keyed on changed-line count).
**Purpose:** Document the system's capabilities, limits, and observed failures
so a team can understand when to trust it — documentation is part of building
reliable agentic systems.

## Summary of changes (one deliberate change per part)
1. **Part 2:** severity validation in `_normalize_issues`.
2. **Part 3:** tightened `should_autofix` policy.
3. **Part 4:** guardrail-backed offline test (`test_single_medium_issue_does_not_autofix`).
4. **Part 5:** completed `model_card.md`.

## Overall purpose of the lab
Learn how to wrap an LLM in a safe agentic workflow: treat the model as one tool,
add rule-based guardrails that validate and risk-score its output, and use tests
to verify the agent acts automatically only when safe and defers to a human
otherwise. The lesson is judgment and reliability, not lines of code.
