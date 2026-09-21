# 🎮 Game Glitch Investigator: The Impossible Guesser

## 🚨 The Situation

An AI was asked to build a simple "Number Guessing Game" using Streamlit. It wrote the
code, ran away, and left behind a game that *looked* finished and was completely broken:

- You couldn't reliably win.
- The hints lied to you.
- The secret number had commitment issues.

None of it crashed on startup, which is what made it dangerous — every bug produced a
confident wrong answer instead of a stack trace. This repo is the repaired version, plus
the paper trail of how it was diagnosed.

## 🛠️ Setup

1. Install dependencies: `pip install -r requirements.txt`
2. Run the app: `python -m streamlit run app.py`
3. Run the tests: `pytest`

## 🎯 The Game's Purpose

A single-player number guessing game. The app picks a secret number inside a range
determined by the selected difficulty, and you have a limited number of guesses to find
it. After each guess the game tells you whether you were too high or too low, and awards
points for winning quickly. It's a small app on purpose — small enough that every bug in
it is a bug you can reason about completely.

| Difficulty | Range | Attempts allowed | Guesses a perfect binary search needs |
|------------|-------|------------------|----------------------------------------|
| Easy | 1–20 | 6 | 5 |
| Normal | 1–100 | 8 | 7 |
| Hard | 1–200 | 8 | 8 |

Every difficulty is winnable with perfect play — that last column is enforced by a test,
because an attempt budget that silently disagrees with its range is exactly the bug that
made the original "impossible".

## 🐛 Bugs Found

Full reproduction steps, expected-vs-actual behaviour, and console output live in
[`reflection.md`](reflection.md) section 1. Summary:

| # | Bug | Root cause |
|---|-----|------------|
| 1 | Hints were backwards — guessing 60 against a secret of 50 said "📈 Go HIGHER!" | `check_guess` returned each outcome paired with the *opposite* advice string |
| 2 | Hints were wrong in a second, sneakier way — a guess of 9 was called too high | `app.py` coerced the secret to `str()` on every even attempt; `guess > secret` raised `TypeError`, and a bare `except TypeError` "recovered" by comparing strings — `"9" > "50"` is `True` |
| 3 | Score went haywire and negative | `update_score` paid "Too High" **+5** on even attempts and **−5** on odd ones, so the reward depended on *when* you guessed; nothing clamped the total |
| 4 | "New Game" never started a new game | The handler reset `attempts` and `secret` but not `score`, `status` or `history`. With `status` still `"won"`, the next rerun hit `st.stop()` — clicking New Game was what kept the app stuck |
| 5 | Changing difficulty made the game unwinnable | The secret was generated once and never reseeded. Switching to Easy (1–20) left the old secret (87) in place. "Hard" was also *narrower* (1–50) than Normal (1–100) |
| 6 | Attempt counting was off by one and punished typos | `attempts` started at `1`, so a fresh Normal game advertised 7 of its 8 attempts; `attempts += 1` ran *before* input validation, so `abc` burned a turn |
| 7 | `pytest` could not run at all | Every function in `logic_utils.py` was a `NotImplementedError` stub, and bare `pytest` failed at collection with `ModuleNotFoundError: No module named 'logic_utils'` |

## 🔧 Fixes Applied

**Refactor first.** All four functions moved out of `app.py` into `logic_utils.py`, which
imports no Streamlit. `app.py` is now UI only: read widgets → call logic → render. That
split is what made the rest testable.

**The structural fix for bug 1.** `check_guess` used to return an `(outcome, message)`
tuple, so the rule and its wording were hand-written side by side and could disagree. It
now returns just an outcome constant (`WIN` / `TOO_HIGH` / `TOO_LOW`), and a new
`hint_message(outcome)` looks the advice up from it. "Too High" can no longer be paired
with "go HIGHER" because nobody writes that pairing by hand any more.

Everything else:

- **Bug 2** — the secret is passed through as the `int` it always was. The bare
  `except TypeError` is gone; `_to_int()` raises loudly on junk instead of falling back
  to a string comparison. A wrong hint is worse than a crash.
- **Bug 3** — both wrong outcomes cost the same fixed 5 points, the total floors at 0,
  and the win bonus is `100 − 10 × (attempts_used − 1)` with a floor of 10, so a
  first-guess win is worth the full 100 (the old formula double-counted the
  already-incremented counter and paid 80).
- **Bug 4** — one `start_new_game()` function resets secret, attempts, score, status and
  history, so no caller can forget a field.
- **Bug 5** — changing difficulty starts a fresh round in the new range. Hard is now
  1–200, genuinely wider than Normal, and two tests check every difficulty is winnable.
- **Bug 6** — `attempts` starts at 0 and is only incremented *after* `parse_guess`
  succeeds, so invalid input is free. The status banner uses the real range and is
  rendered into an `st.empty()` slot at the *end* of the script, so it reflects this
  run's guess instead of lagging a rerun behind.
- **Bug 7** — `conftest.py` and `pytest.ini` put the repo root on `sys.path`, so
  `pytest`, `pytest tests/` and `python -m pytest` all work from any directory.
- **Extra hardening** — `parse_guess` now range-checks the guess, rejects fractional
  input instead of silently truncating `"3.9"` to `3`, and no longer dies with
  `OverflowError` on `"1e999"` (a crash my own fix introduced, caught by a test).

## 📸 Demo Walkthrough

A full game on **Normal** difficulty (range 1–100, 8 attempts, secret = 50, revealed via
the "Developer Debug Info" expander):

1. App loads. The banner reads **"Guess a number between 1 and 100. Attempts left: 8"** —
   all 8 attempts, not 7.
2. User enters `60` → **"📉 Too high -- go LOWER!"**. Attempts left drops to 7, score 0.
3. User enters `abc` → **"That is not a number."** Attempts left stays at **7** — the typo
   is free.
4. User enters `3.9` → **"Whole numbers only -- no decimals."** Still 7 attempts left; the
   game does not quietly score it as a guess of 3.
5. User enters `500` → **"Out of range. Guess between 1 and 100."** Still 7 left.
6. User enters `9` → **"📈 Too low -- go HIGHER!"** — correct, where the original called 9
   *too high* because it was comparing `"9" > "50"` as text. 6 attempts left.
7. User enters `40` → **"📈 Too low -- go HIGHER!"**. 5 attempts left, score still 0 (wrong
   guesses cost 5 but the score is floored at 0 rather than going negative).
8. User enters `50` → 🎈 balloons, **"You won in 5 attempts! The secret was 50. Final
   score: 60"**. The game ends and further guessing is blocked.
9. User clicks **New Game 🔁** → a fresh secret, attempts back to 8, **score back to 0**,
   status back to playing. The original left status at `"won"` here and the app became
   permanently unplayable.
10. User switches difficulty to **Easy** → **"Difficulty changed to Easy. New game
    started."** and the banner becomes **"Guess a number between 1 and 20. Attempts left:
    6"** with a secret reseeded inside 1–20. The original kept the old out-of-range secret
    and still claimed the range was 1–100.
11. Playing Easy and using all 6 attempts without finding it → **"Out of attempts! The
    secret was 7. Score: 0"**, and the game stops until New Game is clicked.

Every step above was replayed against the running app through Streamlit's
`AppTest` harness, not just eyeballed.

## 🧪 Test Results

3 starter tests (kept verbatim) plus 57 added. Each added assertion was also run against
the *original* buggy functions to confirm it actually fails there — a test that passes
against the bug it claims to cover is worthless.

```
$ pytest
============================= test session starts ==============================
platform linux -- Python 3.11.15, pytest-9.0.2, pluggy-1.6.0
rootdir: /home/user/ai110-module1show-gameglitchinvestigator-starter
configfile: pytest.ini
testpaths: tests
collected 60 items

tests/test_game_logic.py ............................................... [ 78%]
.............                                                            [100%]

============================== 60 passed in 0.03s ==============================
```

Verbose, so every test is visible:

```
$ pytest -v
============================= test session starts ==============================
platform linux -- Python 3.11.15, pytest-9.0.2, pluggy-1.6.0 -- /root/.local/share/uv/tools/pytest/bin/python
cachedir: .pytest_cache
rootdir: /home/user/ai110-module1show-gameglitchinvestigator-starter
configfile: pytest.ini
testpaths: tests
collecting ... collected 60 items

tests/test_game_logic.py::test_winning_guess PASSED                      [  1%]
tests/test_game_logic.py::test_guess_too_high PASSED                     [  3%]
tests/test_game_logic.py::test_guess_too_low PASSED                      [  5%]
tests/test_game_logic.py::test_too_high_hint_tells_the_player_to_go_lower PASSED [  6%]
tests/test_game_logic.py::test_too_low_hint_tells_the_player_to_go_higher PASSED [  8%]
tests/test_game_logic.py::test_win_hint_is_congratulations_not_a_direction PASSED [ 10%]
tests/test_game_logic.py::test_comparison_is_numeric_not_lexicographic[9-50-Too Low0] PASSED [ 11%]
tests/test_game_logic.py::test_comparison_is_numeric_not_lexicographic[100-50-Too High0] PASSED [ 13%]
tests/test_game_logic.py::test_comparison_is_numeric_not_lexicographic[9-50-Too Low1] PASSED [ 15%]
tests/test_game_logic.py::test_comparison_is_numeric_not_lexicographic[100-50-Too High1] PASSED [ 16%]
tests/test_game_logic.py::test_win_is_detected_even_when_the_secret_arrives_as_a_string PASSED [ 18%]
tests/test_game_logic.py::test_non_numeric_input_raises_instead_of_guessing PASSED [ 20%]
tests/test_game_logic.py::test_wrong_guesses_score_the_same_regardless_of_attempt_parity PASSED [ 21%]
tests/test_game_logic.py::test_too_high_and_too_low_cost_the_same[1] PASSED [ 23%]
tests/test_game_logic.py::test_too_high_and_too_low_cost_the_same[2] PASSED [ 25%]
tests/test_game_logic.py::test_too_high_and_too_low_cost_the_same[3] PASSED [ 26%]
tests/test_game_logic.py::test_too_high_and_too_low_cost_the_same[4] PASSED [ 28%]
tests/test_game_logic.py::test_wrong_guesses_never_push_the_score_negative PASSED [ 30%]
tests/test_game_logic.py::test_a_first_guess_win_is_worth_full_points PASSED [ 31%]
tests/test_game_logic.py::test_win_points_decay_with_each_attempt_used PASSED [ 33%]
tests/test_game_logic.py::test_unknown_outcome_leaves_the_score_untouched PASSED [ 35%]
tests/test_game_logic.py::test_harder_difficulty_means_a_wider_range PASSED [ 36%]
tests/test_game_logic.py::test_unknown_difficulty_falls_back_to_normal_instead_of_crashing PASSED [ 38%]
tests/test_game_logic.py::test_every_difficulty_is_winnable_by_binary_search[Easy] PASSED [ 40%]
tests/test_game_logic.py::test_every_difficulty_is_winnable_by_binary_search[Hard] PASSED [ 41%]
tests/test_game_logic.py::test_every_difficulty_is_winnable_by_binary_search[Normal] PASSED [ 43%]
tests/test_game_logic.py::test_binary_search_actually_wins_every_secret_in_range[Easy] PASSED [ 45%]
tests/test_game_logic.py::test_binary_search_actually_wins_every_secret_in_range[Hard] PASSED [ 46%]
tests/test_game_logic.py::test_binary_search_actually_wins_every_secret_in_range[Normal] PASSED [ 48%]
tests/test_game_logic.py::test_blank_input_is_rejected[] PASSED          [ 50%]
tests/test_game_logic.py::test_blank_input_is_rejected[   ] PASSED       [ 51%]
tests/test_game_logic.py::test_blank_input_is_rejected[None] PASSED      [ 53%]
tests/test_game_logic.py::test_non_numeric_input_is_rejected[abc] PASSED [ 55%]
tests/test_game_logic.py::test_non_numeric_input_is_rejected[fifty] PASSED [ 56%]
tests/test_game_logic.py::test_non_numeric_input_is_rejected[0x1f] PASSED [ 58%]
tests/test_game_logic.py::test_non_numeric_input_is_rejected[--7] PASSED [ 60%]
tests/test_game_logic.py::test_non_numeric_input_is_rejected[1,000] PASSED [ 61%]
tests/test_game_logic.py::test_non_numeric_input_is_rejected[nan] PASSED [ 63%]
tests/test_game_logic.py::test_full_width_digits_are_accepted_because_python_int_accepts_them PASSED [ 65%]
tests/test_game_logic.py::test_absurdly_large_input_is_rejected_without_crashing[1e999] PASSED [ 66%]
tests/test_game_logic.py::test_absurdly_large_input_is_rejected_without_crashing[-1e999] PASSED [ 68%]
tests/test_game_logic.py::test_surrounding_whitespace_is_tolerated PASSED [ 70%]
tests/test_game_logic.py::test_whitespace_only_input_asks_for_a_guess_rather_than_scolding PASSED [ 71%]
tests/test_game_logic.py::test_fractional_guess_is_rejected_not_silently_truncated PASSED [ 73%]
tests/test_game_logic.py::test_whole_number_written_as_a_decimal_is_accepted PASSED [ 75%]
tests/test_game_logic.py::test_negative_guess_is_accepted_when_no_range_is_supplied PASSED [ 76%]
tests/test_game_logic.py::test_out_of_range_guesses_are_rejected_when_a_range_is_supplied[0] PASSED [ 78%]
tests/test_game_logic.py::test_out_of_range_guesses_are_rejected_when_a_range_is_supplied[101] PASSED [ 80%]
tests/test_game_logic.py::test_out_of_range_guesses_are_rejected_when_a_range_is_supplied[-5] PASSED [ 81%]
tests/test_game_logic.py::test_out_of_range_guesses_are_rejected_when_a_range_is_supplied[999] PASSED [ 83%]
tests/test_game_logic.py::test_guesses_on_and_inside_the_range_boundaries_are_accepted[1] PASSED [ 85%]
tests/test_game_logic.py::test_guesses_on_and_inside_the_range_boundaries_are_accepted[50] PASSED [ 86%]
tests/test_game_logic.py::test_guesses_on_and_inside_the_range_boundaries_are_accepted[100] PASSED [ 88%]
tests/test_game_logic.py::test_minimum_attempts_needed[1-1-1] PASSED     [ 90%]
tests/test_game_logic.py::test_minimum_attempts_needed[1-2-1] PASSED     [ 91%]
tests/test_game_logic.py::test_minimum_attempts_needed[1-8-3] PASSED     [ 93%]
tests/test_game_logic.py::test_minimum_attempts_needed[1-20-5] PASSED    [ 95%]
tests/test_game_logic.py::test_minimum_attempts_needed[1-100-7] PASSED   [ 96%]
tests/test_game_logic.py::test_minimum_attempts_needed[1-200-8] PASSED   [ 98%]
tests/test_game_logic.py::test_minimum_attempts_matches_log2_for_every_difficulty PASSED [100%]

============================== 60 passed in 0.03s ==============================
```

The two tests worth singling out are
`test_binary_search_actually_wins_every_secret_in_range` and
`test_every_difficulty_is_winnable_by_binary_search`. The first plays a complete perfect
game against *every* possible secret in *every* difficulty, navigating only by the hints
`check_guess` returns — if the hints were inverted, the search would never converge. The
second asserts no difficulty can advertise fewer attempts than `ceil(log2(range))`. Between
them they make "the game is impossible" a test failure rather than a bug report.

## 📝 Document Your Experience

- [x] **Describe the game's purpose.** See [The Game's Purpose](#-the-games-purpose).
- [x] **Detail which bugs you found.** See [Bugs Found](#-bugs-found), with full
      reproduction logs in [`reflection.md`](reflection.md) section 1.
- [x] **Explain what fixes you applied.** See [Fixes Applied](#-fixes-applied); every fix
      also carries a `# FIX:` comment at its location in the code.

## 🗂️ Project Layout

```
app.py                     Streamlit UI only -- widgets, session state, rendering
logic_utils.py             All game rules. No Streamlit import, so it is unit testable
conftest.py                Puts the repo root on sys.path so `pytest` can import logic_utils
pytest.ini                 Anchors pytest's rootdir
tests/test_game_logic.py   3 starter tests + 57 regression tests
reflection.md              Bug reproduction log and the AI-collaboration write-up
```

## 🚀 Stretch Features

- [x] **Challenge 1: Advanced Edge-Case Testing** — the suite covers blank and
      whitespace-only input, non-numeric input, fractional and decimal-but-whole guesses,
      negative numbers, range boundaries (1 / 50 / 100 accepted; 0 / 101 / −5 / 999
      rejected), `"1e999"` overflow, full-width Unicode digits, unknown difficulties,
      unknown score outcomes, and an exhaustive binary-search playthrough of every secret
      in every difficulty. Terminal output is in [Test Results](#-test-results) above.
