# BugHound Mini Model Card (Reflection)

Completed after running BugHound in **both** Heuristic and Gemini modes.

---

## 1) What is this system?

**Name:** BugHound

**Purpose:** Analyze a Python snippet, propose a fix, and run reliability checks
before deciding whether the fix should be auto-applied or deferred to a human.

**Intended users:** Students learning agentic workflows and AI reliability concepts.

---

## 2) How does it work?

BugHound runs a small agentic loop:

1. **PLAN** – logs that it is starting a scan + fix-proposal workflow.
2. **ANALYZE** – detects issues. If an LLM client is configured it calls Gemini
   with the analyzer prompt and expects a JSON array of issues; if the key is
   missing, the call errors, or the reply is not parseable JSON, it falls back
   to the offline heuristic analyzer (keyword checks for `print(`, bare
   `except:`, and `TODO`).
3. **ACT** – proposes a fix. With Gemini it uses the fixer prompt ("preserve
   behavior, minimal changes") and strips code fences; offline it does
   mechanical rewrites (`print(` → `logging.info(`, `except:` →
   `except Exception as e:`, prepend `import logging`). No issues → returns the
   original code unchanged.
4. **TEST** – `assess_risk` scores the change 0–100 and assigns low/medium/high.
5. **REFLECT** – decides `should_autofix` and logs whether the fix is safe to
   auto-apply or needs human review.

**Heuristics vs Gemini:** heuristics are deterministic keyword/structural rules;
Gemini provides richer, context-aware analysis and rewrites but can return
malformed output, over-edit, or change behavior — so it is treated as a *tool*
the agent can use, not the final authority. The risk layer is the backstop.

---

## 3) Inputs and outputs

**Inputs:**

- Short Python snippets from `sample_code/`: `cleanish.py` (a clean add
  function), `mixed_issues.py` (TODO + print + bare except in a divide
  function). Shapes: small functions and try/except blocks.

**Outputs:**

- **Issues detected:** Maintainability (TODO), Readability/Code Quality (print),
  Correctness/Reliability (bare except), each with a severity.
- **Fixes proposed:** heuristic mode did line-preserving substitutions; Gemini
  mode rewrote the function, narrowing the except and deleting lines.
- **Risk report:** clean file → LOW / 100 / auto-fix YES; mixed_issues → MEDIUM
  / 45 / auto-fix NO with per-issue reasons plus a "bare except modified" note.

---

## 4) Reliability and safety rules

**Rule A — High-severity deduction (`score -= 40`).**
- Checks: any issue marked "High" severity.
- Why it matters: high-severity issues (e.g. a bare except masking errors)
  indicate the fix touches risky behavior, so trust should drop sharply.
- False positive: an LLM that over-labels a trivial issue as "High" needlessly
  blocks auto-fix.
- False negative: a genuinely dangerous change tagged only "Low"/"Medium"
  escapes the big deduction.

**Rule B — Return-removed check (`"return" in original and not in fixed`).**
- Checks: whether all `return` statements disappeared from the fix.
- Why it matters: silently dropping a return changes the function's output
  contract — a serious behavior change.
- False positive: a legitimate refactor that replaces `return` with `yield` or
  restructures control flow is penalized even though it is correct.
- False negative: it is substring-based, so a fix that keeps *one* `return`
  while removing others passes unpenalized.

---

## 5) Observed failure modes

1. **Format failure / silent fallback (missed correct AI analysis).** On
   `cleanish.py` in Gemini mode the trace logged
   `LLM output was not parseable JSON. Falling back to heuristics.` The model's
   analysis was discarded because it didn't match the expected JSON shape — the
   agent quietly reverted to heuristics with no signal to the user that the AI
   step effectively failed.

2. **Over-editing / behavior change (risky fix).** On `mixed_issues.py` the
   Gemini fixer deleted the `# TODO` comment and the entire `print("computing...")`
   line and narrowed `except:` to `except ZeroDivisionError:`. That is more than
   the minimal change the prompt asked for, and it changes behavior: a
   `TypeError` (e.g. `compute("a", 1)`) now propagates instead of returning 0.

---

## 6) Heuristic vs Gemini comparison

- **Gemini detected** context and rationale heuristics can't — it explained
  *why* the bare except is dangerous (masks `TypeError`) and used richer
  categories (Correctness, Readability).
- **Heuristics caught** the same three patterns consistently and deterministically
  via keywords, with stable severities every run.
- **Fixes differed:** heuristics preserved lines with mechanical substitutions;
  Gemini rewrote and pruned code, sometimes altering behavior.
- **Risk scorer vs intuition:** it agreed — it correctly flagged the
  over-edited mixed_issues fix as MEDIUM / no-autofix, matching the instinct
  that this change needs review.

---

## 7) Human-in-the-loop decision

**Scenario:** a fix that keeps the risk score in the "low" band but still
contains a Medium-severity issue (e.g. a lone TODO), or narrows exception
handling. Auto-applying these silently changes behavior.

- **Trigger added (implemented this lab):** `should_autofix` now requires
  `score >= 90` **and** no Medium/High issues, instead of just `level == "low"`.
  A lone Medium issue (score 80) now defers to a human.
- **Where:** in `reliability/risk_assessor.py` (the risk layer), because it is
  the single authoritative guardrail all modes flow through.
- **Message to user:** "Fix is not safe enough to auto-apply. Human review
  recommended." (already surfaced in the REFLECT trace entry).

---

## 8) Improvement idea

**Add an over-editing guardrail keyed on changed-line count.** The current
"much shorter" rule only fires when the fix drops below 50% of the original
length, so the mixed_issues over-edit (7 → 5 lines) went unpenalized. Adding a
rule that deducts points when the number of changed/removed lines exceeds a
small threshold — with a matching offline test using `MockClient` — would make
the agent more cautious about fixes that quietly touch more code than the issues
require, without adding real complexity.

---

## Changes made during this lab (one per part)

- **Part 2 (analysis reliability):** `_normalize_issues` now validates
  `severity` against {Low, Medium, High} and defaults anything else to Medium,
  so malformed model output can't feed garbage severities into the risk layer.
- **Part 3 (risk policy):** tightened `should_autofix` to `score >= 90` and no
  Medium/High issues, closing the gap where a lone Medium issue auto-applied.
- **Part 4 (guardrail + test):** added `test_single_medium_issue_does_not_autofix`,
  an offline test asserting the tightened policy defers to a human (fails under
  the old policy, passes under the new one). Full suite: 9 passed.
