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

**The rule:** Start with an equally weighted average of the two 0–1 signals; if they strongly disagree, treat the result as uncertain rather than letting one override the other. This is a design decision to test in Milestone 4, not a validated probability.

**The numbers:** `combined = 0.50 × model_score + 0.50 × style_score`. Both scores increase toward AI-like. Proposed thresholds for Milestone 5: human at or below 0.30; unsure above 0.30 and below 0.80; AI at or above 0.80. These differ from the current `config.py` placeholders and must be synchronized when implemented.

**Where it lives:** `scoring.py::combine_signals`

<!-- ⚠️ The grader checks your code against that line. If your rule lives
     somewhere else, say where. -->

**A case where my two signals split, and what my rule does with it:** Hypothetical model score 0.90 and style score 0.10 produce 0.50, which falls in the unsure band. Neither signal alone justifies accusing the writer.



---

## Label Variants

<!-- The exact text of all three labels, as a reader would see them, and the
     score range each covers.

     Write them for someone who has never heard the word "threshold". -->

| Label | Score range | The exact text a reader sees |
|---|---|---|
| high-confidence human | 0.00–0.30 | This writing has patterns more commonly associated with human writing, but the system cannot verify who wrote it. |
| unsure | above 0.30, below 0.80 | We cannot reliably tell whether this writing was produced by a person or AI. This is not an accusation. |
| high-confidence AI | 0.80–1.00 | This writing has patterns associated with AI-generated text. This is not proof of AI use, and you may appeal the result. |

**Why I worded the "unsure" one this way:** A real writer can be flagged because their work is polished or conventional. The label should acknowledge uncertainty without implying misconduct. For example, a human-authored essay might hypothetically receive model_score 0.92, style_score 0.80, combined 0.86, and the AI-like label. The writer should receive a content ID and be able to appeal with a reason and supporting drafts; the appeal should move the item to under_review and be logged. These are illustrative scores, not observed test results.
<!-- It's the hardest of the three. It has to admit uncertainty without
     sounding like an accusation, because the person reading it may well have
     written every word themselves. -->



---

## Endpoint contracts (design; not yet implemented)

**POST /submit** accepts JSON: `text` (required, nonempty string containing writing) and `creator_id` (required, nonempty string identifying the writer). Reject missing, non-string, empty, or oversized text with HTTP 400 or 413 as appropriate; document a maximum before implementation. On success (HTTP 200), return `content_id` (unique string), `guess` (`human`, `unsure`, or `ai`), `confidence` (number 0–1, confidence in the guess, not proof of authorship), `label` (full reader-facing sentence), `model_score` (number 0–1), and `style_score` (number 0–1). The combined score is used internally for label mapping and audit logging. Log the submission using `audit.log_decision`.

**POST /appeal** accepts JSON: `content_id` (required string identifying an existing submission) and `reasoning` (required nonempty string explaining the dispute). Reject invalid fields with HTTP 400 and unknown IDs with HTTP 404. On success, call `audit.log_appeal` to record the dispute and change its status to `under_review`; return HTTP 200 JSON with `content_id` (string), `status` (`under_review`), and `message` (confirmation string). The appeal does not automatically reclassify the writing. Use consistent JSON error fields `error` and `message`.

## Sample Run

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
