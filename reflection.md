# 💭 Reflection: Game Glitch Investigator

Answer each question in 3 to 5 sentences. Be specific and honest about what actually happened while you worked. This is about your process, not trying to sound perfect.

## 1. What was broken when you started?

The first time I ran `python -m streamlit run app.py` the game *looked* finished. The
sidebar had a difficulty picker, the main panel had a guess box, a Submit button, a New
Game button, and a "Developer Debug Info" expander. Nothing crashed on load, which is
exactly what made it dangerous — every bug was a wrong *answer*, not a stack trace. I
opened the debug expander so I could see the secret number, then played on purpose
against a known secret so I could tell the difference between bad luck and bad code.

Six concrete bugs I found:

1. **The hints are backwards.** With the secret at 50, guessing 60 told me
   "📈 Go HIGHER!" and guessing 40 told me "📉 Go LOWER!" In `check_guess`, the branch
   that correctly detects `guess > secret` and labels it `"Too High"` is paired with the
   *"Go HIGHER!"* message, and vice versa. The classification was right; the advice
   attached to it was swapped.
2. **The secret changes type on every other guess.** `app.py` does
   `if st.session_state.attempts % 2 == 0: secret = str(st.session_state.secret)`. On
   even attempts the secret becomes a string, so `guess > secret` raises a `TypeError`,
   which a bare `except TypeError` swallows and "recovers" from by comparing
   `str(guess) > secret` — a *lexicographic* comparison. That is why guessing 9 against a
   secret of 50 was called too high ("9" sorts after "50") and guessing 100 was called too
   low ("100" sorts before "50"). Two bugs stacked: the wrong comparison, then the wrong
   message.
3. **The score goes haywire and goes negative.** `update_score` subtracts 5 for every
   "Too Low" but for "Too High" it adds 5 on even attempts and subtracts 5 on odd ones —
   so the reward depends on *when* you guessed, not *what* you guessed. Four honest
   wrong guesses left me at a score of −15.
4. **"New Game" does not start a new game.** The handler resets `attempts` and `secret`
   but never resets `score`, `status`, or `history`. Because `status` is still `"won"`,
   the very next rerun hits `st.stop()` and prints "You already won. Start a new game to
   play again." — clicking the only button that could fix it. After one win the app is
   permanently unplayable. It also reseeds with a hardcoded `random.randint(1, 100)`,
   ignoring the difficulty range entirely.
5. **Changing difficulty makes the game unwinnable.** The secret is only generated once,
   inside `if "secret" not in st.session_state`. Switching from Normal to Easy narrows the
   range to 1–20 but leaves the old secret (87 in my run) in place, so no legal guess can
   ever match. The info banner is also hardcoded to "between 1 and 100" no matter which
   difficulty is selected — the game lies about its own rules.
6. **Attempt counting is off by one and punishes typos.** `attempts` is initialised to
   `1` instead of `0`, so a fresh Normal game advertises "Attempts left: 7" out of 8
   before you have guessed anything. `attempts += 1` also runs *before* `parse_guess`, so
   typing `abc` burns a turn. And because the banner is rendered earlier in the script
   than the increment, the number shown is always one rerun stale.

On top of that, the starter `pytest` suite failed 3/3: every function in `logic_utils.py`
was a `raise NotImplementedError` stub, and all the real logic lived in `app.py` where no
test could import it without booting Streamlit.

**Bug Reproduction Log**

Reproduced against a pinned secret of **50** on Normal difficulty (secret pinned via the
Developer Debug Info panel / `st.session_state.secret`) unless noted otherwise.

| Input | Expected Behavior | Actual Behavior | Console Output / Error |
|-------|-------------------|-----------------|------------------------|
| Guess of `60`, secret `50` (attempt 1) | Hint "Too High" → advice to go **lower** | Hint shown was "📈 Go HIGHER!" | none (silent wrong answer). Direct call: `check_guess(60, 50) -> ('Too High', '📈 Go HIGHER!')` |
| Guess of `40`, secret `50` (attempt 2) | Hint "Too Low" → advice to go **higher** | Hint shown was "📉 Go LOWER!" | none. Direct call: `check_guess(40, 50) -> ('Too Low', '📉 Go LOWER!')` |
| Guess of `9`, secret `50`, on an **even** attempt | "Too Low" → go higher | "📈 Go HIGHER!" — 9 was classified as *too high* | none, because `except TypeError` hides it. Raw comparison: `TypeError: '>' not supported between instances of 'int' and 'str'`; then `"9" > "50" -> True` |
| Guess of `100`, secret `50`, on an **even** attempt | "Too High" → go lower | "📉 Go LOWER!" — 100 was classified as *too low* | none. `"100" > "50" -> False` (lexicographic) |
| Four wrong guesses `40, 30, 20, 10` against secret `50` | Score never drops below 0 | Score ran `-5, -10, -15, -20` | none. `update_score(0,"Too Low",1) -> -5` |
| Guess of `60` (Too High) on attempt 2, then on attempt 3 | Same outcome scores the same either way | `+5` on attempt 2, `-5` on attempt 3 | none. `update_score(0,"Too High",2) -> 5` vs `update_score(0,"Too High",3) -> -5` |
| Win, then click **New Game 🔁** | Fresh game: score 0, attempts 0, playable | `score` stayed `60`, `status` stayed `"won"`; page showed "You already won. Start a new game to play again." and `st.stop()` fired — app permanently stuck | none |
| Start on Normal (secret `87`), switch difficulty to **Easy** (range 1–20), guess `10`, `20`, `5` | Secret reseeded into 1–20; game winnable | Secret stayed `87`; every guess returned "📉 Go LOWER!"; banner still read "Guess a number between 1 and 100" | none |
| Page load, Normal difficulty, before any guess | "Attempts left: 8" | "Attempts left: 7" | none (`attempts` initialised to `1`) |
| Guess of `abc` | Error message, attempt **not** consumed | "That is not a number." shown, but `attempts` advanced `1 → 2` | `That is not a number.` (handled, no traceback) |
| `python -m pytest tests/` on the starter code | 3 passed | 3 failed | `NotImplementedError: Refactor this function from app.py into logic_utils.py` |

---

## 2. How did you use AI as a teammate?

- Which AI tools did you use on this project (for example: ChatGPT, Gemini, Copilot)?
- Give one example of an AI suggestion that was correct (including what the AI suggested and how you verified the result).
- Give one example of an AI suggestion you did not accept as written (including what the AI suggested, why you rejected or changed it, and how you verified your version). It does not have to be a suggestion that was wrong: over-engineered, out of scope, harder to read, or a poor fit for this codebase all count.

---

## 3. Debugging and testing your fixes

- How did you decide whether a bug was really fixed?
- Describe at least one test you ran (manual or using pytest)  
  and what it showed you about your code.
- Did AI help you design or understand any tests? How?

---

## 4. What did you learn about Streamlit and state?

- How would you explain Streamlit "reruns" and session state to a friend who has never used Streamlit?

---

## 5. Looking ahead: your developer habits

- What is one habit or strategy from this project that you want to reuse in future labs or projects?
  - This could be a testing habit, a prompting strategy, or a way you used Git.
- What is one thing you would do differently next time you work with AI on a coding task?
- In one or two sentences, describe how this project changed the way you think about AI generated code.
