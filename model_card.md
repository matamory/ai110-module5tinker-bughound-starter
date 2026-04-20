# BugHound Mini Model Card (Reflection)

Fill this out after you run BugHound in **both** modes (Heuristic and Gemini).

---

## 1) What is this system?

**Name:** BugHound  
**Purpose:** Analyze a Python snippet, propose a fix, and run reliability checks before suggesting whether the fix should be auto-applied.

**Intended users:** Students learning agentic workflows and AI reliability concepts.

---

## 2) How does it work?

BugHound follows a plan -> analyze -> act -> test -> reflect loop. It starts by planning a quick scan-and-fix pass, then analyzes code using heuristics in offline mode (for patterns like `print(`, bare `except:`, and `TODO`) or Gemini in API mode. If Gemini output fails to parse or the API fails, analysis falls back to heuristics. In the act step, heuristics apply small text/regex edits, while Gemini is asked for full rewritten code; if Gemini returns empty/error output, the agent falls back to the heuristic fixer. The test step runs `assess_risk`, and the reflect step allows auto-fix only for low-risk results.

---

## 3) Inputs and outputs

**Inputs:**

I tested `sample_code/cleanish.py`, `sample_code/flaky_try_except.py`, and `sample_code/mixed_issues.py`, plus a weird case with comments only: `# TODO: handle edge cases`. These inputs covered short functions, `try/except` control-flow blocks, logging/print patterns, and comments-only text.

**Outputs:**

The agent produced issues in Code Quality, Reliability, and Maintainability with Low/Medium/High severities. Proposed fixes included adding logging imports, replacing `print(...)` with `logging.info(...)`, and replacing bare `except:` with `except Exception as e:`. The risk report returned `score`, `level`, `reasons`, and `should_autofix`; observed outcomes included high risk for mixed issues and medium risk for the TODO-only no-op case after the guardrail update.

---

## 4) Reliability and safety rules

One core rule applies severity penalties (High -40, Medium -20, Low -5), which is necessary because severe findings should reduce trust quickly; however, it can produce false positives if severity is overstated and false negatives if risky edits are labeled low severity. Another rule penalizes missing returns when `return` exists before but not after, which helps catch behavior-breaking changes; however, it can flag valid refactors and still miss subtle return-value changes when `return` is still present. A new guardrail also penalizes cases where issues were detected but the proposed fix made no code changes, which prevents unsafe auto-fix confidence; this can still false-positive on informational-only issues and false-negative when a small but incorrect change is made.

---

## 5) Observed failure modes

A repeated failure mode was Gemini analyzer format mismatch: on `sample_code/flaky_try_except.py` and `sample_code/mixed_issues.py`, logs showed unparseable JSON and analysis fell back to heuristics. Another failure mode was Gemini fixer empty output on `sample_code/mixed_issues.py`, which forced fallback fixing and reduced confidence in AI-generated edits. A third reliability problem was unsafe confidence in no-op fixes: with TODO-only input, the system detected an issue but previously could still mark the result as auto-fix safe until the new guardrail was added.

---

## 6) Heuristic vs Gemini comparison

In these runs, heuristics were more reliable for issue detection because Gemini analyzer output was often not parseable and fell back. Heuristics consistently caught `print`, bare `except:`, and `TODO` in the tested snippets, while Gemini-mode fixes frequently ended up using heuristic fallback because model output was empty or invalid. Risk scoring generally matched intuition for mixed issues (high risk, no auto-fix). The TODO-only no-op case was initially too permissive, but after the guardrail it moved to medium risk with auto-fix disabled.

---

## 7) Human-in-the-loop decision

BugHound should require human review when issues are detected but the proposed fix is unchanged or nearly unchanged. The trigger should be a policy check in `risk_assessor` that blocks auto-fix when issues exist and fixed code equals original code. The user-facing message should say that issues were detected but no meaningful change was applied, so human review is required.

---

## 8) Improvement idea

A low-complexity improvement is to enforce one strict analyzer schema, such as requiring Gemini to return `{ "issues": [...] }` only. The parser should validate that schema first and then perform a single fallback to heuristics if validation fails. A focused test should assert that malformed Gemini output triggers fallback with a parse-failure log. This would reduce ambiguous parser behavior and make failures easier to diagnose.
