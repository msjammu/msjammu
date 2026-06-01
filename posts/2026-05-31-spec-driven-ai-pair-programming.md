# Shipping a Real Community Tool in an Afternoon with Spec-Driven AI Pair-Programming

*A field report on using GitHub Copilot's agent mode to take a community problem from a vague description to a deployed tool — with the spec doing most of the heavy lifting.*

**Project:** [Blood Drive Pre-Screening Tool](https://punjabischoolbothell.org/programs/blood-drive/) for the Sikh Center of Seattle (built for [Punjabi School Bothell](https://punjabischoolbothell.org/))
**Case study:** [Reducing Wasted Slots at a Community Blood Drive](./blood-drive-case-study.md)
**Stack:** One static HTML file. No build step. No framework. No analytics.

---

## TL;DR

Our community blood drive kept losing appointment slots to donors who were ineligible on the day of donation — most often because of recent international travel. In an afternoon, working with GitHub Copilot in agent mode, we:

1. Wrote a short spec describing the problem and the smallest useful solution.
2. Pruned the spec down once we found the partner ([Bloodworks Northwest](https://bloodworksnw.org/eligibility-checker)) already publishes a full eligibility checklist.
3. Generated a single-file static web tool that pre-screens donors in 30 seconds.
4. Iterated five or six times — adding short-circuit logic, dynamic sub-questions, a Back button, fixing a real bug, deep-linking to the right drive.
5. Wrote an automated end-to-end test, ran 24 assertions, fixed the test (not the code), and shipped.

The whole thing is one HTML file, served from the PSB site at <https://punjabischoolbothell.org/programs/blood-drive/>.

The interesting part isn't the code — it's the **workflow**. This piece is about that workflow, the rules that made it work, and the places where the spec earned its keep.

## Why a spec at all?

The temptation with an AI coding assistant is to skip straight to "build me X." That works for throwaway snippets. It does not work when:

- The problem is fuzzy and the right scope isn't obvious.
- You're going to iterate ten times and need a stable reference.
- Someone other than you will eventually look at this.
- You want to push back on the model when it over-builds.

A short spec — even one written in five minutes — gives you all four. It becomes the contract you and the AI keep returning to. When the model wants to add a country picker, you point at the line that says "keep it bare minimum" and it complies. When you change your mind, you edit the spec first, then the code. The spec is the slow-changing artifact; the code is downstream.

## The flow we actually used

### 1. Voice-drafted the problem out loud

The first message to Copilot was effectively a stream-of-consciousness description of the problem: the Sikh Center of Seattle hosts a community blood drive, parents travel back to home countries, they show up and get deferred, slots get wasted — *can we build something*? No requirements doc. No structure. Just the situation.

Copilot's first response was a structured proposal: a welcome screen, hard-stop questions, additional screening, Green/Yellow/Red outcomes. That became the seed of the spec.

**Lesson:** Start unstructured. Let the assistant impose the first structure. Then react to it.

### 2. Wrote a v0.1 spec

We turned the proposal into a `SPEC.md` with sections for Problem, Goals, Non-Goals, Users, User Flow, Functional Requirements, Tech Stack, Rollout, Open Questions. Standard PRD-lite.

It was too ambitious. It had a country-by-country travel rule engine, optional AI chat, multi-language support, analytics, a rules JSON authored by a volunteer.

That was fine — drafts are allowed to be wrong. The next step is what mattered.

### 3. Killed scope with one external fact

Before building anything, we checked what the partner already publishes. Bloodworks Northwest hosts a [60-Second Eligibility Checklist](https://bloodworksnw.org/eligibility-checker) with the full ruleset.

That single fact collapsed half the spec:

> "We should keep the checker tool to bare minimum to ensure this is quick and effective."

Out went the country picker, the medication list, the rules JSON, the analytics. The spec shrank to **five questions** that catch the most common community-specific reasons for deferral, and a "Yellow" off-ramp that says *go run the official checker*.

**Lesson:** Before building, find out what the world already has. The biggest scope wins come from not building something.

### 4. Generated the implementation

The new, leaner spec said: single static HTML file, mobile-first, no build, ships on the existing GitHub Pages site, config in a JSON-shaped block at the top.

Copilot generated the whole page in one shot. Bootstrap 5 + FontAwesome to match the rest of the site. A small `CONFIG` object for drive details, a `QUESTIONS` array for the question data, and a tiny engine that walks the questions and renders a result.

Because the spec was clear about constraints — *no PII, no analytics, no build step, mobile-first, match site styling* — there was no debate about any of these. The model just complied.

### 5. Iterated against the spec, not the code

Once the page existed, every change was a one-line user request that we mapped to a spec property:

| User request | Spec property it maps to |
|---|---|
| "At each step jump to red if the check fails" | UX speed — don't waste a clear-fail donor's time |
| "Add sub-questions under travel if Yes" | Specificity for the highest-frequency reason |
| "Back should return to the last page" | Forgive misclicks; reduce friction |
| "Remove address, keep time" | Bare minimum on the welcome screen |
| "The appointment slot should be ..." | Sikh-Center-specific drive deep link |

Each request became a small diff. The spec made it obvious where each change belonged.

### 6. Hit a real bug — and we found it because we clicked

The "Back" button worked from any normal question. But after a Red short-circuit on Question 1, Back jumped to Question 5 instead of Question 1.

The cause: Red short-circuit set `state.step = queue.length`, so the Back handler's naive `step - 1` landed at the end. The fix was to derive the previous step from `history.length` (the true count of answered questions) instead of subtracting from `state.step`. Three lines.

Two things to call out here:

1. **The user found the bug, not the AI.** Copilot was happy to claim the feature worked. A human clicking around in 10 seconds caught it.
2. **The fix was small because the model preserved the abstraction.** `state.history` already existed for un-splicing follow-up questions; we just used it as the source of truth for navigation too.

**Lesson:** AI coding assistants are excellent at writing the next change. They are *not* a substitute for clicking through your own UI.

### 7. End-to-end test, ran the suite, fixed the test

Before pushing, we asked Copilot to write and run an end-to-end test using the browser tooling. It generated a Playwright script that covered ten user journeys: Green, Yellow, Red on each hard-stop position, the travel sub-chain with all three malaria answers, Back after a Red short-circuit, mid-flow Back un-splicing follow-ups, and Start Over.

First run: 17 of 24 assertions passed. Seven failures all looked similar — substring checks on the progress counter.

We didn't change the code. We looked at the failures and noticed the assertions were case-sensitive (`Question 1 of 5`) but the CSS applied `text-transform: uppercase`, so `innerText` returned `QUESTION 1 OF 5`. Lowercase the comparisons. Re-ran. **24 of 24 passed.**

**Lesson:** Failing tests aren't always code bugs. Read the failure before reaching for the fix. The model is great at writing test scaffolding but doesn't always reason about CSS-affected DOM strings.

### 8. Site integration, then ship

Last steps were trivial because the structure held:

- Added a **Community** dropdown to every root page's nav.
- Renamed `SPEC.md` to `README.md` so GitHub renders it as the folder landing page.
- Reframed the README as a case study (background → problem → constraints → solution → value → out of scope → post-drive follow-up).
- Single commit, push, done.

Total elapsed time: an afternoon. Total lines of net new code: ~500.

## Patterns that worked

### "Keep it bare minimum" as an explicit rule

The single most valuable line in the spec was a literal sentence: *"Design rule: keep it bare minimum. Anything not high-frequency for our community is routed to Bloodworks rather than re-implemented here."*

We quoted this back to the model — implicitly — every time it offered to add complexity. The rule was a fence the model respected.

### Spec-first, code-second, on every change

Even small UX requests started with a spec lens: *"is this in scope per the spec?"* If yes, change the code. If no, change the spec first or push back on the request.

This prevents scope creep from being a thousand individual yeses. Each yes is fine; the cumulative drift is not.

### Config-first design

Putting the drive date, time, scheduler URL, Bloodworks links, and phone in a single `CONFIG` object at the top of the script made one thing true: **a volunteer can update the tool for the next drive without touching engine code.** That single design choice probably matters more than any other line in the file.

### Reading the model's diffs

We treated every AI-produced diff like a junior dev's PR — read it, understand it, accept or push back. A few times the model added a helper or an abstraction we didn't need; calling that out (and citing the implementation-discipline principle) kept the file lean.

### Stopping when done

There's always one more feature ("Add a QR code generator! Add Punjabi! Add analytics!"). We listed them as Open Questions in the README and shipped. The drive is on June 21. The best version of this tool is the one that's live before then.

## Patterns that didn't work (and what we did instead)

### "Trust the model on UI correctness" — don't

We trusted the first Back-button implementation. It looked right in the code. Real clicks revealed the Q1 short-circuit bug.

**Fix:** every UI change gets a manual click-through before it ships. The model can write the test; the human still has to drive the browser.

### "Let the model find the right scope on its own" — don't

The first spec was too ambitious because nothing in the conversation pushed back. Once we added the constraint *"keep it bare minimum"* and the fact *"Bloodworks already publishes a full checker"*, the model converged on the right scope immediately.

**Fix:** Constraints are more powerful than goals. Add constraints early.

### "One giant test at the end" — almost don't

The end-to-end test was a great idea, but writing it after every feature was already in would have been brittle. Next time: write a tiny smoke test after the first green-path renders, then add assertions per feature.

## What the spec did for us, concretely

Without the spec we would have:

- Built a country picker we didn't need.
- Reimplemented half of Bloodworks' rules and had to maintain them.
- Stored donor responses "for analytics" without thinking about PII.
- Skipped the disclaimer.
- Probably forgotten to handle the Back-after-Red case at all.

With the spec we:

- Said no to country pickers in seconds, not days.
- Pointed at Bloodworks for everything past the bare minimum.
- Made "no PII, no storage" a non-negotiable from line one.
- Wrote the disclaimer copy once and reused it on every screen.
- Caught the Back-after-Red bug because the spec had stated *"Back should be on the last page where user was"* — making the requirement testable.

## Reusable workflow for the next project

1. **Talk to the assistant in plain language about the problem.** No structure. See what shape it proposes.
2. **Write a one-page spec in markdown.** Problem, goal, non-goals, constraints, design rule, success metric. That's it.
3. **Check what already exists.** A 30-second search for an existing partner tool or library can delete half your spec.
4. **Add explicit constraints to the spec.** "Keep it bare minimum." "Single file." "No PII." "No build step." Constraints are how you steer.
5. **Generate the first version in one shot.** Don't piecemeal it. A complete first cut is easier to critique.
6. **Iterate via small user requests mapped back to spec properties.** Push back when a request would violate the spec.
7. **Click through your own UI before declaring victory.** Every time.
8. **Have the assistant write an end-to-end test, then run it and read the failures honestly.**
9. **Ship the README as a case study.** Future-you and your team will both thank you.

## Why this matters beyond one tool

This was a 500-line static page for a community blood drive. The same workflow applies to a 5,000-line internal service or a customer-facing feature.

What changes at scale is the cost of *not* having the spec. Without one, an AI-paired afternoon is a great way to ship a brittle pile of features that nobody can extend. With one, the same afternoon produces a coherent thing with a clear scope, a real boundary, and a story you can tell.

The model is fast. The spec is the steering wheel.

---

### About this project

- **Live tool:** <https://punjabischoolbothell.org/programs/blood-drive/>
- **Case study:** [Reducing Wasted Slots at a Community Blood Drive](./blood-drive-case-study.md)
- **Partner:** [Bloodworks Northwest](https://bloodworksnw.org/)
- **Drive:** Sikh Center of Seattle — Sunday, June 21, 2026

If you run community drives and want to copy this pattern, the entire tool is one HTML file. Fork it, change the `CONFIG` block, change the five questions for your community's reality, point the schedule link at your drive, and ship.

---

*Author: [Manpreet Jammu](https://github.com/msjammu) · [LinkedIn](https://www.linkedin.com/in/msjammu/)*
