# Google AI Harness extra credit: testing the Team 11 `select_seat` browser agent

**Name:** Mengyu Wu  
**Agent tested:** the Team 11 Agent Airways agent from Part 2 (`agent_one_team_tool.py` with `team_tools/select_seat.py`, both unchanged)  
**Model:** qwen3:8b through Ollama, run in Google Colab with a GPU runtime  
**Request used in every run:** "Select seat 1A for our pretend reservation."

## Step 1. How Google's idea of a harness relates to our browser agent

A harness is everything around the model that turns its text into reliable behavior: the tools it may call, the checks on its output, the loop that runs actions, the memory of earlier steps, and the test of whether the job is done. Our browser agent has exactly this shape. The language model only *proposes* one action as JSON, for example `{"action":"select_seat","argument":{"seat":"1A"}}`. Python then checks the proposal (`validate`), runs the tool in a real Playwright browser, reads the page back, and sends that observation to the model for its next decision. Python also decides when the run may stop: the model cannot end the task as a success unless a tool has verified it. So the reliability comes mostly from the Python harness, not from the model alone.

## Step 2. How the program decides the task succeeded

Success is checked in two layers, both in Python.

1. **The tool checks the browser** (`team_tools/select_seat.py`). After clicking the requested seat, `select_seat` reads three separate facts from the page and returns `ok: true` only if all three hold:
   - exactly one seat button has the `selected` class and its `data-seat` equals the requested seat;
   - the seat message on the page is exactly `Selected seat 1A.`;
   - the Continue button is enabled.

   Before clicking it also refuses a bad argument (`validate`: one key, `seat`, in a format like `1A`), a page that is not the Agent Airways practice site, an ambiguous or missing seat screen, and a disabled or unavailable seat.
2. **The runner blocks unsupported success claims** (`agent_one_team_tool.py`). If the model says `done` with status `success`, the runner accepts it only when the most recent `select_seat` call returned `ok: true`. Otherwise it answers "Success blocked: no verified successful team-tool call" and the run continues.

I also checked each run myself from the saved evidence: `stop_reason` is `success`, `used_team_tool` and `last_team_tool_ok` are both `true`, and the screenshot shows seat 1A highlighted, the text "Selected seat 1A." and an enabled Continue button.

What this does **not** check: the runner does not compare the selected seat with the customer's original request. If the model had asked for another seat, the tool would verify *that* seat. In these runs the model asked for 1A each time, so the result is correct, but the harness would not catch a wrong-but-valid seat.

## Step 3. Five runs

Each run was a separate call of the same command, with only the `--label` changed so that evidence files were not overwritten:

```
python agent_one_team_tool.py --tool select_seat --case success --headless --label run1
```

Each run starts a new headless Chromium browser and loads the practice site from scratch (the runner's own setup step, `prepare_page`), so no page state carries over between runs. The program was not edited between runs.

| Run | Result | Notes |
|---|---|---|
| 1 | Success | Step 1: `select_seat` 1A, tool returned `ok: true` ("The browser confirms seat 1A is selected and Continue is enabled."). Step 2: model said `done`, status `success`. 2 steps. |
| 2 | Success | Same two steps. The model's final wording differed slightly ("Seat 1A has been selected successfully."). |
| 3 | Success | Same two steps and same wording as run 1. |
| 4 | Success | Same two steps and same wording as run 1. |
| 5 | Success | Same two steps and same wording as run 1. |

**Evidence** (all in this folder):

- Logs: [logs/run1.log](logs/run1.log) to [logs/run5.log](logs/run5.log)
- Runner records (model proposals, tool observations, final page text): `evidence/select_seat_run1_agent.json` to `evidence/select_seat_run5_agent.json`
- Final screenshots: `evidence/select_seat_run1_final.png` to `evidence/select_seat_run5_final.png`

## Step 4. What I learned

All five runs succeeded. In every run the model made the same two moves: it requested `select_seat` with seat 1A, the tool confirmed the selection in the browser, and the model then reported success. The only difference between runs was a few words in the model's final sentence (run 2), which shows that the model's wording can vary while the checked outcome stays the same. What I learned is that the test of "did it work" does not rely on the model's sentence at all: it relies on the browser readback, and the runner would have blocked a success claim without it.

**Limitations of this test.** Five identical runs of one easy request are weak evidence of reliability. The request was the same each time, the target seat was available each time, and the site never changed, so the agent was never pushed into a failure, a refusal, or a recovery. The model's answers were also nearly identical, so the five runs behave almost like one run repeated. A stronger test would vary the request (other seats, an unavailable seat such as 1B, a malformed seat name), run many more times, and include a deliberate fault, such as a slow or changed page, to see whether the harness still catches wrong results. The earlier Part 2 runs of this same agent (unavailable seat 1B, and 1B with fallback to 2A) cover some of those cases but were not part of these five.
