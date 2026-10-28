# Provenance Guard

> ### 👋 Start here
>
> **New to this repo? Read [RUNNING.md](RUNNING.md) first.**
>
> Once `python test.py` passes, in one terminal:
>
> ```bash
> python app.py
> ```
>
> and in another:
>
> ```bash
> curl -X POST http://127.0.0.1:5000/ping \
>   -H "Content-Type: application/json" \
>   -d '{"message": "hello"}'
> ```
>
> That's a web service. Something sent it text and it sent something back.
> Everything this unit is filling in the middle.
>
> **The rest of this file is your submission.** Fill it in as you go.

---

<!-- ─────────────────────────────────────────────────────────────────────────
     Unit 7 asks for the first five sections. Unit 8 adds the five below.

     Everything is pasted as TEXT. A pasted curl response gets full credit;
     a picture of your terminal gets none.
     ───────────────────────────────────────────────────────────────────────── -->

<!-- ═══════════════════════ UNIT 7 — THE BUILD ═══════════════════════ -->

## What This Does

1. A writer sends JSON containing `text` and `creator_id` to `POST /submit`.
2. The Flask route validates the request and assigns a unique `content_id`, so a decision can be traced and appealed.
3. `detector.py::model_signal` runs the provided local language model and returns a 0–1 predictability-based AI-like score.
4. `stylometry.py::style_signal` measures sentence-length spread, vocabulary repetition, and punctuation patterns, producing a separate 0–1 AI-like score.
5. `scoring.py::combine_signals` combines those scores. Disagreement should reduce certainty, not become an accusation.
6. `scoring.py::score_to_label` maps the combined score to human, unsure, or AI, using explicit thresholds and reader-friendly language.
7. `audit.log_decision` records the decision and both signals; the API sends the `content_id`, scores, guess, confidence, and label back to the reader. A writer can challenge it through `POST /appeal`.

A score describes patterns in text; it does not prove who wrote it. This is the proposed design, not a claim that the unfinished routes already work.

---

## Signals and Scoring

<!-- For each signal: what property it measures, why that might differ between
     human and AI writing, and WHAT IT CAN'T SEE.

     The blind spot is not optional. If you can't name it you don't understand
     the signal yet — and unit 8's attack set is built out of blind spots. -->

### Signal one — the model signal

**What it measures:** Predictability of the submitted text under the local language model, converted to an AI-like score. It is not a direct authorship observation.

**Why that might differ between human and AI writing:** Generated text can favor probable next words and conventional sequences, making some passages easier for a model to predict.

**What it can't see:** The writing history, drafts, intent, or whether a human deliberately wrote predictable prose. Edited or paraphrased AI text may also evade the pattern.

### Signal two — the style signal

**What it measures:** The shape of writing: normalized variation in sentence length, type-token ratio (unique words divided by total words), and punctuation density. The planned `style_signal` converts these to a 0–1 AI-like score.

**Why that might differ between human and AI writing:** Some generated passages use similarly sized sentences and repetitive, conventional phrasing, while human prose may vary more. These are tendencies, not rules.

**What it can't see:** Meaning, authorship, writing history, or the difference between deliberate stylistic consistency and automation. Typos, inserted punctuation, quotations, and short passages can distort its measurements. Vocabulary ratio also changes with text length.

### The combining rule

<!-- Write the rule in words, then give the numbers. What happens when the two
     signals disagree? -->

**The rule:** Compute an equal-weight average of the two 0–1 signals. If their absolute difference is at least 0.45, move the combined score halfway toward 0.50 to represent disagreement and reduce overconfident classifications. Implemented in Milestone 4; not yet calibrated on a verified dataset.

**The numbers:** `base = 0.50 × model_score + 0.50 × style_score`; if `abs(model_score - style_score) >= 0.45`, use `combined = 0.50 + 0.50 × (base - 0.50)`; otherwise `combined = base`. Weights are in `config.py`. Implemented Milestone 5 thresholds: human at or below 0.30; unsure above 0.30 and below 0.80; AI-like at or above 0.80.

**Where it lives:** `scoring.py::combine_signals`

<!-- ⚠️ The grader checks your code against that line. If your rule lives
     somewhere else, say where. -->

**A case where my two signals split, and what my rule does with it:** Hypothetical model score 0.90 and style score 0.10 give a base of 0.50; disagreement is 0.80, so the moderation rule leaves the combined score at 0.50. This is an illustrative split, not yet a measured one.



---

### Milestone 4 — implementation and calibration plan

The style signal now uses three normalized features: (1) sentence-length coefficient of variation, with low spread considered more AI-like (weight 0.40); (2) unique-word ratio from the first 50 words, with lower diversity considered more AI-like (weight 0.40); and (3) comma/semicolon/colon/question/exclamation/dash marks per word, with fewer marks considered more AI-like (weight 0.20). Normalization caps are 0.75, 0.90, and 0.12 respectively. These are **heuristic choices**, not empirical accuracy estimates. A one-sentence passage has zero measured spread, and short text is unreliable.

The `/submit` response and `audit.log_decision` now include `model_score`, `style_score`, and `combined_score`. `guess`, `confidence`, and the reader label are now computed in Milestone 5. The new API response still needs local verification; the sample below documents the older Milestone 4 version.

**Observed three-passage style calibration (local terminal):**

| Passage | Model score (prior test) | Style score | Combined score (computed with current rule) |
|---|---:|---:|---:|
| AI-style | 0.8404 | 0.3589 | 0.5498 |
| Human-style | 0.5684 | 0.5063 | 0.5373 |
| In-between | 0.5968 | 0.4538 | 0.5253 |

The first passage provides an **observed disagreement**: the model score is 0.8404 while the style score is 0.3589, a difference of 0.4815. The initial average is 0.59965; disagreement moderation moves it to 0.549825 (rounded to 0.5498). The combined numbers are calculated from the observed component scores, **not yet observed from a live updated API response**. The first style test reported vocabulary ratio 0.9118 and punctuation density 0.0882; the second reported sentence spread 0.125, vocabulary ratio 0.8438 and punctuation density 0.0312; the third reported sentence spread 0.1538, vocabulary ratio 0.9231 and punctuation density 0.0385. The first test's sentence spread was not provided.

**Observed Milestone 4 API response (local `POST /submit`):**

```json
{
  "combined_score": 0.5373,
  "confidence": 0.0,
  "content_id": "4fb26dad-1d9b-43fc-a123-71e2db0e5208",
  "guess": "unsure",
  "label": "Pending final label mapping; both signals have been evaluated.",
  "model_score": 0.5684,
  "style_score": 0.5063
}
```

This confirms both signals and the combined score appear in the response. The average is 0.53735, rounded by Python to 0.5373. The final guess, confidence and label remain Milestone 5 placeholders. An audit-log entry matching this content ID has not yet been verified.

**Verified human-written reflection — observed disagreement (same `$humanText` variable):**

The author supplied their own reflection on sports drinks and electrolytes and ran both command-line detectors on the same `$humanText` variable. The model reported perplexity **12.5** and model signal **0.8864**. Stylometry reported sentence-length spread **0.3214**, type-token ratio **0.9091**, punctuation density **0.0000**, and style signal **0.4286**. Their difference is **0.4578**, above the disagreement threshold of 0.45. The unadjusted average is **0.6575**, and the disagreement-adjusted score is **0.57875** (the final four-decimal representation should be verified by running `scoring.combine_signals` locally). This is an observed false-positive **risk** from the model signal on self-reported human writing, not a validated classification or a live `/submit` result.

**Local testing still required:** Run `python stylometry.py "your passage"` on the same three passages used with `detector.py`. Then run both signals on four deliberately chosen passages: clearly AI, genuinely human-authored, formal human-authored, and lightly edited AI-generated text. Record the **actual** model, style, and combined scores for each. Identify one observed disagreement and discuss which signal likely caused it. Do not label a generated example as verified human writing.

```powershell
git pull origin main
python stylometry.py "The implementation of this system provides several important benefits. First, it improves efficiency and consistency. Second, it allows users to process information more effectively. Finally, it offers a reliable approach to completing common tasks."
python stylometry.py "Yesterday I missed the bus because I stopped to help my neighbor pick up a bag of oranges. I got to class late, with sticky fingers and a funny story to tell."
python stylometry.py "The project was interesting because it helped me learn about technology. Some parts were difficult, but I worked through the problems and eventually finished the assignment."
```

## Label Variants

<!-- The exact text of all three labels, as a reader would see them, and the
     score range each covers.

     Write them for someone who has never heard the word "threshold". -->

| Label | Score range | The exact text a reader sees |
|---|---|---|
| high-confidence human | 0.00–0.30 (inclusive) | We think this was probably written by a person, but this result is not proof of authorship. |
| unsure | above 0.30, below 0.80 | We cannot reliably tell whether a person or AI wrote this text, so no authorship conclusion should be drawn. |
| high-confidence AI | 0.80–1.00 (inclusive) | This writing has patterns often found in AI-generated text, but this result cannot establish who wrote it. |

These are the exact strings returned by `scoring.py::score_to_label`. The high-confidence wording refers to score bands, **not verified authorship**. Run `python -c "import scoring; print(scoring.label_ranges())"` to inspect ranges. To verify all branches without relying on model behavior, run `python -c "from scoring import score_to_label; print([score_to_label(x) for x in (0.2, 0.5, 0.9)])"`. These are mapping unit checks, not three actual text submissions.

**Why I worded the "unsure" one this way:** A real writer can be flagged because their work is polished or conventional. The label should acknowledge uncertainty without implying misconduct. For example, a human-authored essay might hypothetically receive model_score 0.92, style_score 0.80, combined 0.86, and the AI-like label. The writer should receive a content ID and be able to appeal with a reason and supporting drafts; the appeal should move the item to under_review and be logged. These are illustrative scores, not observed test results.
<!-- It's the hardest of the three. It has to admit uncertainty without
     sounding like an accusation, because the person reading it may well have
     written every word themselves. -->



---

## Endpoint contracts (implemented; local Milestone 5 testing pending)

**POST /submit** accepts JSON: `text` (required, nonempty string containing writing) and `creator_id` (required, nonempty string identifying the writer). Reject missing, non-string, empty, or oversized text with HTTP 400 or 413 as appropriate; document a maximum before implementation. On success (HTTP 200), return `content_id` (unique string), `guess` (`human`, `unsure`, or `ai`), `confidence` (number 0–1, confidence in the guess, not proof of authorship), `label` (full reader-facing sentence), `model_score` (number 0–1), and `style_score` (number 0–1). The combined score is used internally for label mapping and audit logging. Log the submission using `audit.log_decision`.

**POST /appeal** accepts JSON: `content_id` (required string identifying an existing submission) and `reasoning` (required nonempty string explaining the dispute). Reject invalid fields with HTTP 400 and unknown IDs with HTTP 404. On success, call `audit.log_appeal` to record the dispute and change its status to `under_review`; return HTTP 200 JSON with `content_id` (string), `status` (`under_review`), and `message` (confirmation string). The appeal does not automatically reclassify the writing. Use consistent JSON error fields `error` and `message`.

## Milestone 3 verification (run locally)

The `/submit` route is implemented with **one** signal. The following are commands to run, **not recorded results**. Do not substitute invented scores or logs for actual output.

**Step 1 — independently inspect the detector on three passages:**

```bash
python detector.py "The following paragraph was generated by an AI system for testing. The service receives a passage, computes a score, and returns the score. This paragraph repeats a straightforward structure and uses conventional wording."
python detector.py "Yesterday I missed the bus because I stopped to help my neighbor pick up a bag of oranges. I got to class late, with sticky fingers and a funny story to tell."
python detector.py "The report describes several useful observations. Some results were expected, but others surprised the group, so we decided to check the measurements again before drawing conclusions."
```

The first is an **AI-written test example**; the other two are **illustrative sample passages**, not verified human-authored or borderline ground truth. For the required calibration, replace them with a passage you personally wrote and a passage you judge borderline. Record the three actual `perplexity` and `model_signal` outputs here after running them.

**Observed detector calibration (user's local terminal):**

| Passage | Perplexity | Model signal (higher = more AI-like) |
|---|---:|---:|
| AI-style sample | 15.9 | 0.8404 |
| Human-style sample | 37.9 | 0.5684 |
| In-between sample | 35.2 | 0.5968 |

These are real `python detector.py` outputs from three sample passages. They are not proof of authorship or independently verified classification accuracy. The terminal also printed a nonfatal `loss_type=None` warning and used the default `ForCausalLMLoss`.

**Initial API test (unsuccessful; PowerShell quoting issue):**

```json
{"error": "invalid_json", "message": "Expected a JSON object."}
```

The malformed `curl.exe` command also printed multiple `Could not resolve host` errors. A subsequent `GET /log?limit=3` returned:

```json
{"count": 0, "entries": []}
```

This is expected for an invalid request: no successful submission had been logged. **A successful submission response and matching audit entry have not yet been supplied.** On Windows PowerShell, retry with `Invoke-RestMethod` and `ConvertTo-Json` to avoid shell-quoting errors.

**Step 2 — start the service and send a real submission:**

```bash
python app.py
```

In a second terminal:

```bash
curl -X POST http://127.0.0.1:5000/submit \
  -H "Content-Type: application/json" \
  -d '{"text":"Yesterday I missed the bus because I stopped to help my neighbor pick up a bag of oranges. I got to class late, with sticky fingers and a funny story to tell.","creator_id":"milestone3-test"}'
curl "http://127.0.0.1:5000/log?limit=3"
```

**What to verify:** `/submit` returns HTTP 200 with a nonempty `content_id`, numeric `model_score`, placeholder `guess`/`confidence`/`label`, and `style_score: null`. The `/log` response must contain an entry with the **same** `content_id`, `creator_id`, `model_score`, `status: "decided"`, and a timestamp. The actual outputs still need to be pasted into the Sample Run section below.

**Testing limitation:** The intermediate hardcoded-response-then-curl checkpoint and live model tests were not performed in this environment; they should not be reported as passed.

### Milestone 5 — local verification still needed

The label mapping and `POST /appeal` route have been implemented and committed, but **the actual updated `/submit`, `/appeal`, and `GET /log` responses have not yet been supplied from the user's local server**. Do not present examples as observed output. Appeals are appended to the audit log with the same `content_id`, `event: "appeal"`, `status: "under_review"`, and the writer's `reasoning`; the original decision is preserved. The latest status for the content ID is therefore `under_review`.

After pulling and restarting `python app.py`, run in a second PowerShell terminal:

```powershell
$body = @{text="Yesterday I missed the bus because I stopped to help my neighbor pick up a bag of oranges. I got to class late, with sticky fingers and a funny story to tell."; creator_id="milestone5-test"} | ConvertTo-Json
$submission = Invoke-RestMethod "http://127.0.0.1:5000/submit" -Method Post -ContentType "application/json" -Body $body
$submission | ConvertTo-Json
$appealBody = @{content_id=$submission.content_id; reasoning="I wrote this myself and can provide my drafts."} | ConvertTo-Json
Invoke-RestMethod "http://127.0.0.1:5000/appeal" -Method Post -ContentType "application/json" -Body $appealBody | ConvertTo-Json
(Invoke-RestMethod "http://127.0.0.1:5000/log").entries | Where-Object { $_.content_id -eq $submission.content_id } | ConvertTo-Json -Depth 5
```

Paste the **real** submission, appeal, and matching log output below in Sample Run after testing. Also submit real text producing all three labels. If the extreme labels are difficult to reach, report that honestly; `label_ranges()` describes mapping bands, not actual model output reachability.

### Milestone 5 — observed submission and appeal

These responses were copied from the user's local PowerShell run after updating the Milestone 5 code. The original submission and appeal share the same real `content_id`. **The matching `GET /log` output has not yet been supplied or verified.**

**Observed `POST /submit` response:**

```json
{
  "combined_score": 0.5373,
  "confidence": 0.0746,
  "content_id": "6da8c519-01f4-4bb6-8ebd-62f35cf92e6c",
  "guess": "unsure",
  "label": "We cannot reliably tell whether a person or AI wrote this text, so no authorship conclusion should be drawn.",
  "model_score": 0.5684,
  "style_score": 0.5063
}
```

**Observed `POST /appeal` response:**

```json
{
  "content_id": "6da8c519-01f4-4bb6-8ebd-62f35cf92e6c",
  "message": "Your appeal has been recorded and is awaiting review.",
  "status": "under_review"
}
```

The label is the reader-facing `unsure` variant; the appeal response confirms the status change. The user supplied this observed appeal audit-log entry (the original `decided` entry was not included in the pasted output):

```json
{
  "content_id": "6da8c519-01f4-4bb6-8ebd-62f35cf92e6c",
  "creator_id": "milestone5-test",
  "event": "appeal",
  "reasoning": "I wrote this myself and can provide my drafts.",
  "status": "under_review",
  "timestamp": "2026-10-28T23:18:43+00:00"
}
```

This confirms the appeal event is persisted under the submitted content ID. The original `decided` audit entry was subsequently supplied from the local log and matches the same `content_id`. The complete observed decision-to-appeal audit trail is:

```json
[
  {
    "combined_score": 0.5373,
    "content_id": "6da8c519-01f4-4bb6-8ebd-62f35cf92e6c",
    "creator_id": "milestone5-test",
    "guess": "unsure",
    "label": "We cannot reliably tell whether a person or AI wrote this text, so no authorship conclusion should be drawn.",
    "model_score": 0.5684,
    "status": "decided",
    "style_score": 0.5063,
    "timestamp": "2026-10-28T23:18:17+00:00"
  },
  {
    "content_id": "6da8c519-01f4-4bb6-8ebd-62f35cf92e6c",
    "creator_id": "milestone5-test",
    "event": "appeal",
    "reasoning": "I wrote this myself and can provide my drafts.",
    "status": "under_review",
    "timestamp": "2026-10-28T23:18:43+00:00"
  }
]
```

Both records were independently supplied from the user's local `GET /log` output. The append-only log retains the original decision while recording the appeal under the same ID.

### Milestone 5 — observed live label-reachability tests

Four actual text submissions to the local Flask service produced the following outputs (higher scores are more AI-like):

| Test passage | Model score | Style score | Combined score | Actual guess |
|---|---:|---:|---:|---|
| AI-style | 0.8404 | 0.3589 | 0.5498 | `unsure` |
| Human-style (illustrative, not verified authorship) | 0.5684 | 0.5063 | 0.5373 | `unsure` |
| Repetitive | 0.9872 | 0.9259 | 0.9566 | `ai` |
| Varied | 0.4684 | 0.1754 | 0.3219 | `unsure` |

These are the user's real `POST /submit` results, not predicted scores. The `ai` and `unsure` branches were reached in this first set. The human branch was subsequently reached in a second set of tests below. No thresholds were changed to force a passing result.

**Additional observed live submissions:**

| Test passage | Model score | Style score | Combined score | Actual guess |
|---|---:|---:|---:|---|
| Personal story | 0.5905 | 0.1693 | 0.3799 | `unsure` |
| Unusual vocabulary | 0.2245 | 0.2162 | 0.2203 | `human` |
| Conversational | 0.6387 | 0.0156 | 0.4136 | `unsure` |

**All three label branches have now been observed from real text submissions:** `human` (unusual vocabulary, 0.2203), `unsure` (human-style illustrative passage, 0.5373), and `ai` (repetitive passage, 0.9566). These labels indicate model behavior, not independently verified text authorship.

## Sample Run

### Milestone 3 — verified responses

Submission command (PowerShell):

```powershell
$body = @{
  text = "Yesterday I missed the bus because I stopped to help my neighbor pick up a bag of oranges. I got to class late, with sticky fingers and a funny story to tell."
  creator_id = "malakai"
} | ConvertTo-Json
Invoke-RestMethod -Uri "http://127.0.0.1:5000/submit" -Method Post -ContentType "application/json" -Body $body | ConvertTo-Json
```

Actual submission response:

```json
{"confidence":0.0,"content_id":"00cc65e5-9010-438c-b06a-1bed9124b282","guess":"unsure","label":"Pending full evaluation; only the model signal has run.","model_score":0.5684,"style_score":null}
```

Log command:

```powershell
Invoke-RestMethod "http://127.0.0.1:5000/log?limit=5" | ConvertTo-Json -Depth 5
```

Actual log response:

```json
{
  "count": 2,
  "entries": [
    {"combined_score":null,"content_id":"817b6df8-7985-4620-9700-fbaabb122217","creator_id":"malakai","guess":"unsure","label":"Pending full evaluation; only the model signal has run.","model_score":0.5684,"status":"decided","style_score":null,"timestamp":"2026-10-28T22:51:58+00:00"},
    {"combined_score":null,"content_id":"00cc65e5-9010-438c-b06a-1bed9124b282","creator_id":"malakai","guess":"unsure","label":"Pending full evaluation; only the model signal has run.","model_score":0.5684,"status":"decided","style_score":null,"timestamp":"2026-10-28T22:52:28+00:00"}
  ]
}
```

The second audit entry matches the submitted content ID. The second signal, combined score, final label, and appeal route are for later milestones.


<!-- One submission and one appeal, pasted as text: the request, the response,
     and the log entries. -->

**A submission**

<!-- The response needs content_id, guess, confidence, label, AND both
     signal scores - model_score and style_score. Unit 8's tools read those
     two off the response, so leaving them out costs you next unit's
     diagnostics. -->

```bash
$ curl -X POST http://127.0.0.1:5000/submit \
  -H "Content-Type: application/json" \
  -d '{"text": "...", "creator_id": "..."}'
```

```json

```

**An appeal**

```bash
$ curl -X POST http://127.0.0.1:5000/appeal \
  -H "Content-Type: application/json" \
  -d '{"content_id": "...", "reasoning": "..."}'
```

```json

```

**What the log shows afterwards**

```bash
$ curl "http://127.0.0.1:5000/log?limit=3"
```

```json

```

---

## How I Used AI

**Moment 1**

- *What I asked for:*
- *What came back:*
- *What I changed:*

**Moment 2**

- *What I asked for:*
- *What came back:*
- *What I changed:*

<!-- ═══════════════════════ UNIT 8 — THE TEST ═══════════════════════ -->

---

## Rate Limiting

**My limits:** ___ per minute, ___ per day

**Why those numbers and not others:**
<!-- Think about two people: a real writer submitting their own work a few
     times an hour, and a script sending a thousand variations to map your
     thresholds. Your numbers have to be liveable for the first and hostile to
     the second. -->

**What a caller counts as:** <!-- per address, or per creator_id? They fail
differently — one script can look like a thousand callers, and one household
can look like one. -->

**The run of status codes when I pushed past it:**

<!-- python run_attacks.py --flood 15 -->

```

```

---

## Attack Run — Before

<!-- Every attack with its outcome and a held-or-broke mark.
     `run_attacks.py --label before` produces the table. -->



**Ten audit log entries from the run**

<!-- Choose entries that show the interesting failures, not the ten easiest. -->

```json

```

**The single worst outcome** — the one I'd least want to explain to a writer
whose work got caught by it:



---

## Run Log and Verdicts

<!-- Your five criteria across three trials. `run_eval.py --label before`. -->

| Criterion | Target | Run 1 | Run 2 | Run 3 | Verdict |
|---|---|---|---|---|---|
| 1.  |  |  |  |  |  |
| 2.  |  |  |  |  |  |
| 3.  |  |  |  |  |  |
| 4.  |  |  |  |  |  |
| 5.  |  |  |  |  |  |

**Real output from one trial**, pasted as text, naming the file and function
that produced it:

```json

```

**Diagnoses**

<!-- For each miss AND each broken attack: the stage and the mechanism.

     Your service has five stages: the route, signal one, signal two, the
     combining rule, and the label mapping.

     Not a diagnosis:  "The evasion attacks got through."
     A diagnosis:      "Four evasion inputs got through. All four had typos
                        added to AI-generated text. My second signal measures
                        word variety, and typos raise word variety — so it read
                        them as more human. My combining rule weights that
                        signal at 0.6, so it dragged the whole score down."

     Look for a pattern. Five failures on one signal is one problem, not five. -->



---

## The Improvement

**What I changed:**

**Which diagnosis pointed at it:**

### Attack Run — After

<!-- Same format. `run_attacks.py --label after` -->



**Did it help, and how do I know:**

<!-- Widening the unsure band usually fixes one problem and creates another.
     Reporting that trade honestly is worth full credit. -->



---

## What's Still Broken

<!-- For each attack still getting through and each criterion still missed:
     what you'd do, and why you stopped. -->



**The trade I'd make if this ran for real:**

<!-- Two sentences. Fewer wrong accusations means more AI text getting through.
     Say which way you'd go and WHO PAYS FOR IT. -->



<!-- ═════════════════════════════════════════════════════════════════════

     SUBMISSION CHECKLIST — unit 7

       [ ] criteria.md has five numbered criteria, each naming a target
       [ ] Each has a reason underneath
       [ ] POST /submit returns content_id, guess, confidence, label,
           model_score and style_score
       [ ] Two signals that measure DIFFERENT things, each with a named blind spot
       [ ] Three label variants, all reachable, written out in full
       [ ] POST /appeal changes status and writes to the log
       [ ] Signals and Scoring names scoring.py::combine_signals
       [ ] Sample Run: a submission AND an appeal, as pasted text
       [ ] At least four new commits
       [ ] Repository URL submitted — WRITE IT DOWN

     SUBMISSION CHECKLIST — unit 8

       [ ] Rate limiting on /submit, with your numbers justified
       [ ] The pasted run of status codes showing 429
       [ ] Audit log with all nine fields, recording rejections too
       [ ] The attack set run IN FULL, every input with an outcome
       [ ] Ten audit log entries pasted as text
       [ ] Five criteria across three trials
       [ ] A diagnosis for every miss and every broken attack — stage AND mechanism
       [ ] One improvement, with Attack Run — After
       [ ] What's Still Broken, including the production trade
       [ ] At least four new commits
       [ ] The SAME repository URL as last unit

     Do not delete and recreate this repository.
     ═════════════════════════════════════════════════════════════════════ -->

---

📖 **How to run this project: [RUNNING.md](RUNNING.md)**
