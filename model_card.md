# BugHound Mini Model Card (Reflection)

---

## 1) What is this system?

**Name:** BugHound  
**Purpose:** Analyze a Python snippet, propose a fix, and run reliability checks before suggesting whether the fix should be auto-applied.

**Intended users:** Students learning agentic workflows and AI reliability concepts.

**Modes:** Heuristic only (offline, uses `MockClient`), Gemini (API key required), and Ollama (local `gemma4`, no key).

---

## 2) How does it work?

BugHound runs five steps in `BugHoundAgent.run`:

1. **Plan:** logs the intended scan + fix workflow.
2. **Analyze:** finds issues. With no usable LLM (or `MockClient`), regex/substring heuristics run (`print(`, bare `except:`, `TODO`). With an LLM, the analyzer prompt asks for a JSON array of issues.
3. **Act:** proposes a fix. Heuristics replace `print(` with `logging.info(` (adding `import logging`) and rewrite bare `except:`. With an LLM, the fixer prompt returns rewritten code, with code fences stripped.
4. **Test:** `assess_risk` scores the fix from 0 to 100 using issue severity and structural checks.
5. **Reflect:** `should_autofix` is true only when the risk level is `low` (score >= 75).

**Fallbacks.** The agent falls back to heuristics when the LLM call raises, returns output that is not parseable JSON, or returns an empty fix.

**Parsing hardening (changed).** `_parse_json_array_of_issues` and `_normalize_issues` now:

- drop issues with no `msg`
- map unknown or missing severities to `Medium` instead of `"Unknown"`
- treat a non-empty array with no usable issues as unparseable, which triggers the heuristic fallback

---

## 3) Inputs and outputs

**Inputs:**

- Short Python snippets: small functions with `print` calls, bare `except:` blocks, and `TODO` comments.
- The tests use `def f():\n    print('hi')\n    return True\n`.

**Outputs:**

- Issues detected: Code Quality (print), Reliability (bare except), Maintainability (TODO). LLM modes can report others.
- Fixes proposed: logging replacement, `except Exception as e:` rewrite. There is no heuristic fix for TODOs.
- Risk report: `score`, `level`, `reasons`, `should_autofix`.


---

## 4) Reliability and safety rules

Rules in `assess_risk`:

| Rule | What it checks | Why it matters | False positive | False negative |
|---|---|---|---|---|
| Severity deductions (high -40, medium -20, low -5) | Penalizes by reported issue severity | More serious issues mean a riskier change | A trivial fix to a "High" issue is still penalized heavily | An LLM that under-reports severity avoids the penalty |
| Length check | Fixed code under 50% of original lines | Large deletions often mean lost logic | A legitimate cleanup that removes dead code | A same-length rewrite that changes behavior |
| `return` removed | `"return"` in original but not in fixed | Removing returns changes behavior | Matches the word in comments or strings | Changing what is returned goes unnoticed |
| Bare `except:` modified | `"except:"` in original but not in fixed | Exception handling changes can alter control flow | The intended fix is penalized | Other handler changes are unchecked |
| Empty fix | Empty output gives score 0 | Nothing usable to apply | None likely | None |

**Change made:** the bare-except penalty went from **-5 to -15**. The old value was too small for a change the code comment itself calls risky. The bare-except heuristic reports a High issue (-40), and the fix then adds -15, so such fixes now land in the medium/high range and need review.

---

## 5) Observed failure modes


1. **Missed issues:** the heuristics only look for three patterns. Anything else, such as mutable default args, off-by-one errors, or unclosed files, is missed. A valid LLM response of `[]` is also accepted as "no issues", so a model that misses a real bug looks like a clean result.
2. **Risky or wrong fixes:**
   - The heuristic bare-except rewrite hard-codes 8-space indentation, so it can produce invalid code at other nesting depths.
   - `.replace("print(", ...)` also rewrites `pprint(` and strings containing `print(`.
   - `import logging` is prepended above module docstrings or `from __future__` imports.
   - LLM fixes are never checked to be valid Python, and prose-wrapped output without code fences is treated as code.

---

## 6) Heuristic vs Gemini comparison

- What did the LLMs detect that heuristics did not?
- What did heuristics catch consistently?
- How did the proposed fixes differ?
- Did the risk scorer agree with your intuition?

**What is verified:** with `MockClient`, analysis always falls back to heuristics because it returns non-JSON for analyzer prompts. Tests cover six malformed analyzer outputs: prose, empty string, unterminated array, JSON object, array of non-dicts, and an issue missing `msg`. Each falls back correctly.

---

## 7) Human-in-the-loop decision

**Scenario:** the fix removes or rewrites a large part of the code, or changes exception handling.

- **Trigger:** require review when the fixed code fails `ast.parse`, or when any High-severity issue is present, regardless of score.
- **Where:** `risk_assessor.py`, as a hard override of `should_autofix`, so it applies to every client.
- **Message:** "This fix changes error handling or may alter behavior. Review the diff before applying."

---

## 8) Improvement idea

Add an `ast.parse` check on `fixed_code` in `assess_risk`. If the fix does not parse, force score 0. This catches broken heuristic bare-except rewrites and LLM outputs that are prose or truncated, with only a few lines of code and a simple test.

---

## Changes made in this iteration

- **Risk assessor:** bare-except penalty raised from -5 to -15.
- **Parsing:** stricter issue normalization (see section 2).
- **Tests:** a parametrized test covering six non-JSON analyzer outputs, all falling back to heuristics.
- **Ollama support:** new `OllamaClient` in `llm_client.py` and an "Ollama (local gemma4)" mode in the app. Errors return an empty string, which triggers the heuristic fallback. `import ollama` is at module top level, so the package is now required even for Heuristic mode.
