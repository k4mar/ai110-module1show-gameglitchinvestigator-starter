# AI Interactions Log

> **Stretch features only.** Only the sections below correspond to stretch work I actually
> attempted. Sections for stretch features I did not attempt (SF9 Linting & Style, SF11
> Model Comparison) have been removed rather than left as empty templates.

---

## Agent Workflow (SF8)

> Document your experience using an AI agent (e.g., Cursor Agent, Claude, Copilot) to make
> multi-step changes autonomously.

**What task did you give the agent?**

Claude Code in agent mode, with `app.py`, `logic_utils.py` and `tests/test_game_logic.py`
in context. The instruction was deliberately multi-step rather than a single edit:

> "Move `check_guess`, `parse_guess`, `get_range_for_difficulty` and `update_score` from
> `app.py` into `logic_utils.py`, fix the inverted high/low hint, remove the
> string-comparison fallback, and update the imports in `app.py` so the UI file has no
> game logic left in it."

**What did the agent do?**

- Moved all four functions into `logic_utils.py` and replaced the `NotImplementedError`
  stubs, keeping the docstrings that were already there.
- Rewrote `check_guess` to return a bare outcome constant instead of an
  `(outcome, message)` tuple, and added `hint_message(outcome)` to derive the wording — a
  design change I had not asked for, and the best idea in the session (see `reflection.md`
  section 2).
- Replaced the `attempt_limit_map` dict in `app.py` with a single `DIFFICULTIES` table in
  `logic_utils.py`, so a difficulty's range and its attempt budget can't drift apart.
- Rewrote the import block in `app.py` and removed the now-dead function bodies.
- Consolidated the state resets into one `start_new_game()` function.

**What did you have to verify or fix manually?**

- **Rejected its bug-2 approach.** Its first pass made `check_guess` silently coerce a
  stringified secret with `int(str(secret))`. That hides a type error in the logic layer —
  the same mistake as the original `except TypeError`. I kept the root fix in `app.py`
  (stop stringifying the secret) and made `_to_int()` raise loudly on non-numeric input.
- **Fixed a crash it introduced.** Its `parse_guess` rewrite caught only `ValueError`, but
  `float("1e999")` is `inf` and `int(inf)` raises `OverflowError`, which would have taken
  the page down. Added a `math.isinf` / `math.isnan` guard.
- **Corrected a false factual claim.** It documented a `.strip()` fix as being needed
  because `" 42"` failed to parse. Python's `int()` already strips whitespace, so that had
  never been broken. Corrected the comment and the test.
- **Finished a fix it only claimed.** The stale "Attempts left" banner was still stale
  after its edits, because the banner renders near the top of the script and the guess is
  processed further down. I rewrote it using an `st.empty()` slot filled at the end of the
  script.
- **Fixed a syntax error in its output.** Its `conftest.py` had an unterminated
  triple-quoted docstring; `pytest` refused to collect until I closed it.
- Reviewed every hunk of the diff before keeping it, and re-ran the full suite plus a
  scripted `AppTest` replay of the whole bug log after each change.

---

## Test Generation (SF7)

> Document how you used AI to help generate or improve tests.

Prompting note: asking for "tests for this function" produced tests that described the
code's current shape. Asking for **"pytest cases that would fail against the *old* buggy
`check_guess`"** produced tests about the *bugs*, which is what I actually wanted. Every
row below was verified by running the assertion against the original buggy implementation
— an assertion that passes against the bug it claims to cover is worthless.

| Edge Case | Prompt Used | AI-Suggested Test | Did It Pass? | Your Reasoning |
|-----------|-------------|-------------------|--------------|----------------|
| Hint direction for a too-high guess | "Write a pytest case that fails against the old `check_guess`, where 60 vs secret 50 returned ('Too High', 'Go HIGHER!')" | Assert `"LOWER" in hint_message(check_guess(60, 50))` and `"HIGHER" not in` it | ✅ Passes now; **fails** against old code | Kept as written. The negative half of the assertion matters — without it, a message containing both words would slip through. |
| Lexicographic vs. numeric comparison | "The secret is sometimes a str. Generate cases where a string compare and a numeric compare disagree." | Parametrised `(9, "50") -> TOO_LOW` and `(100, "50") -> TOO_HIGH` | ✅ Passes now; **fails** against old code | Kept. These are the two exact inputs from my bug log; `"9" > "50"` and `"100" < "50"` are both true as text and both wrong as numbers. |
| Score parity bug | "Write a test showing Too High and Too Low cost the same" | `assert update_score(50, TOO_HIGH, 1) == update_score(50, TOO_LOW, 1)` | ✅ Passed — **but also passed against the buggy code** | **Rewrote.** The old code charged −5 for both at attempt 1, so this test caught nothing. The parity bug only appears on even attempts. Parametrised over attempts 1–4 and left a comment saying the parameter is load-bearing. |
| Score floor | "Can the score go negative?" | Loop four wrong guesses, assert `score >= 0` after each | ✅ Passes now; **fails** against old code (reaches −20) | Kept. Asserting inside the loop rather than only at the end tells me *which* attempt broke the invariant. |
| First-guess win value | "Is the win bonus correct?" | `assert update_score(0, WIN, 1) == 100` | ✅ Passes now; **fails** against old code (returns 80) | Kept. Exposed an off-by-two: the old formula was `100 - 10 * (attempt_number + 1)` *and* `app.py` had already incremented the counter. |
| Blank vs. whitespace-only input | "Enumerate every falsy-ish input `parse_guess` should reject" | Parametrise `["", "   ", None]`, all expecting `"Enter a guess."` | ✅ Passes now; the `"   "` case **fails** against old code | Kept. Good catch by the AI: the original only tested `raw == ""`, so `"   "` fell through to `int("   ")` and got the wrong error message. |
| Surrounding whitespace | (AI volunteered: "the original never calls `.strip()`, so `' 42'` fails") | `assert parse_guess(" 42 ") == (True, 42, None)` | ✅ Passed — **but also passed against the buggy code** | **Demoted.** The AI's premise was false: Python's `int()` already strips whitespace, so this was never a bug. Kept as an explicitly-labelled guard, not a regression test, and corrected the code comment it had talked me into writing. |
| Fractional guess | "What does `parse_guess('3.9')` do and is that right?" | Assert `"3.9"` is rejected rather than truncated | ✅ Passes now; **fails** against old code (silently returns 3) | Kept, and added the inverse case `"42.0" -> 42`, which the AI had not suggested — rejecting decimals outright would have been too blunt. |
| Full-width Unicode digits | "Any non-obvious non-numeric strings worth testing?" | Assert `"4２"` is rejected as not a number | ❌ **Failed** | **The test was wrong, not the code.** `int()` accepts any Unicode decimal digit, so `int("4２") == 42`. Rather than adding code to reject it, I inverted the assertion and kept it as `test_full_width_digits_are_accepted_because_python_int_accepts_them`, with a docstring recording the mistaken assumption. Deleting it would have erased the lesson. |
| Huge exponent input | (Mine, not the AI's) | `parse_guess("1e999")` should be rejected, not crash | ❌ **Failed** initially | Found a crash **the AI's own fix had introduced**: `float("1e999")` is `inf`, `int(inf)` raises `OverflowError`, and the function only caught `ValueError`. Added a `math.isinf` / `math.isnan` guard. |
| Every difficulty actually winnable | (Mine, prompted by the AI's `DIFFICULTIES` refactor) | — | ✅ Passes | The AI's idea of one table for range + attempts made me realise they're a single coupled decision. Wrote `test_every_difficulty_is_winnable_by_binary_search` (asserting `ceil(log2(range)) <= attempt_limit`) and `test_binary_search_actually_wins_every_secret_in_range`, which plays a full perfect game against every secret in every difficulty navigating only by `check_guess` hints. Inverted hints would make it never converge. |

**Outcome:** 3 starter tests kept verbatim and passing, 57 added, 60 total. 10 of 10
regression assertions confirmed to fail against the original buggy implementations.
