# Acceptance criteria — Provenance Guard

Five criteria that say what "working" means for this service, written in unit 7
**before** you built it.

**All five are yours.** None are given.

An acceptance criterion names a number or something plainly observable. *"The
detector is accurate"* is an opinion — and perfect AI detection isn't possible
anyway, so it's an opinion about something nobody can deliver. *"Of 10 pieces I
wrote myself, at most 1 is labelled high-confidence AI"* is a criterion.

Under each, write a sentence or two on **why that target**. *"I allow at most 1
false positive in 10 because on a writing site a wrong accusation costs more
than a missed detection"* is a real answer.

> Missing your own targets next unit costs you nothing. Setting a target so
> easy you can't miss it does.

---

## Pick numbers you can defend

**Cover at least three of these five areas.** They're prompts, not a form.

| Area | A question it could answer |
|---|---|
| False positives | How often may a human's writing get called AI? |
| Score spread | How far apart should clearly-AI and clearly-human text score? |
| Label coverage | Is every one of your three labels actually reachable? |
| The appeal path | Does an appeal always change status and always get recorded? |
| Bad input | What happens when `text` is missing, empty, or enormous? |

Two things worth knowing before you pick numbers:

- **The expensive mistake is one-sided.** Missing some AI text costs the site a
  little. Accusing a real writer costs that person a lot. A criterion that
  treats both errors as equally bad is describing a system nobody would ship.
- **Unit 8 tests across three trials, and the target has to hold across all
  three.** Your first signal runs a model, so identical input can score
  differently twice. A target of "at most 1 in 10" against trials of 2, 1, 2 is
  a **miss**.

---
## 1. False positives on human writing

Across each of three evaluation trials, of 10 independently verified human-written passages, **at most 1 of 10** may receive the **high-confidence AI** label. Use the same 10 passages per trial and record each individual label.

**Why this target:** A false accusation can harm a real writer more than a missed AI-generated passage harms the service. Allowing no more than one high-confidence false positive in ten is a demanding, checkable starting target rather than a promise of perfect detection.

---

## 2. Score separation between human and AI examples

In **each of three trials**, the median combined AI-likelihood score of 10 clearly AI-generated passages must exceed the median score of 10 independently verified human-written passages by **at least 0.20** on the 0–1 scale. Keep the two sets fixed across trials.

**Why this target:** If the two groups score nearly the same, the combined signal is not meaningfully separating them. A 0.20 median gap requires a noticeable difference while allowing individual passages to overlap.

---

## 3. All three labels are reachable

In **each of three trials**, a fixed test set containing at least **20 diverse submissions** must produce **at least one** instance of each of the three labels: **high-confidence human**, **unsure**, and **high-confidence AI**. Count labels from actual `POST /submit` responses, not manually supplied scores.

**Why this target:** A label that exists in code but never appears to readers is not useful. Requiring all three from real requests also checks that the score combination and threshold mapping work together.

---

## 4. Appeals change status and leave an audit trail

In **each of three trials**, submit **5 valid appeals** for 5 existing, distinct `content_id` values. **All 5 of 5** must receive a successful response, change the submission status to `under_review`, and create a corresponding appeal entry in the audit log containing the matching `content_id` and reasoning. Also submit **one unknown `content_id`** per trial; it must return **HTTP 404** without creating a valid appeal.

**Why this target:** A person wrongly labeled AI needs a reliable way to challenge the decision. Recording every valid appeal makes the process reviewable, while rejecting unknown IDs prevents meaningless or misleading appeal records.

---

## 5. Invalid input is rejected without crashing

In **each of three trials**, test these **5 invalid `POST /submit` requests**: missing `text`, empty `text`, whitespace-only `text`, numeric `text`, and text longer than **20,000 characters**. **All 5 of 5** must return a JSON error with a 4xx status (400 for malformed fields; 413 for oversized text), and **none** may return HTTP 500 or a successful classification. A valid submission immediately afterward must still receive a normal response.

**Why this target:** Bad input should not crash the service or generate a misleading authorship verdict. A 20,000-character ceiling is a concrete resource safeguard for a local model, and testing recovery afterward checks that an error does not break later requests.

---

<!-- ─────────────────────────────────────────────────────────────────────────
     UNIT 8 — read this before you change anything above.

     If a criterion turns out to be BROKEN rather than merely unmet, you can
     revise it, and that earns credit. But never delete or edit the original
     line. Add the revision underneath, like this:

         ## 2. Uncertain cases land in the unsure band

         The system handles uncertain cases well.

         **Why this target:** ...

         > **Revised in unit 8:** Text I judged borderline scores between
         > 0.4 and 0.6 in at least 4 of 5 cases.
         >
         > **Why revised:** "well" gave me nothing to check. I couldn't
         > reach a verdict from it at all.

     That's a revision because the criterion couldn't be MEASURED.

     Lowering a target because you missed it is not a revision, and it costs
     you the point:

         ✗ "At most 1 of 10 of my own pieces flagged as AI" → "at most 3
            of 10", because 1 of 10 turned out to be hard.

     A number you missed stays where it is, gets diagnosed, and gets a fix
     attempted. That's where the points are.
     ───────────────────────────────────────────────────────────────────────── -->
