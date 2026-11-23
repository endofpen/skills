---
name: implement
description: Workflow step "implement" – the user roughly describes a feature or change; first stress-test the plan with critical questions in rounds until it is solid, get explicit approval of the plan, then implement it in Kotlin without committing and finish with a short German commit message. Use for "implement", "implementieren", "umsetzen", "build this feature", "neues Feature".
---

# Implement

This skill takes a rough idea from the user to working production code. It has four phases. **Never skip a phase and never move on without the condition at the end of each phase being met.**

1. Understand – read the description and the relevant code
2. Grill – critical questions in rounds until the plan is solid
3. Plan – present the plan, get explicit approval
4. Implement – write the code, verify it, write a commit message

Communicate with the user in the language they use (usually German). Code, identifiers, and code comments follow the conventions of the repository.

The questioning approach is adapted from Matt Pocock's `grill-me` / `grilling` skills (github.com/mattpocock/skills, MIT).

---

## Phase 1 – Understand

1. If the user invoked the skill without a description, ask one open question: what do they want to build, and why? Wait for the answer.
2. Check the branch state: `git status`, current branch name. If the working tree has unrelated uncommitted changes, point this out before continuing.
3. **Explore the codebase** for everything the description touches: affected classes, their callers, existing similar features, data model, API endpoints, events, configuration, existing conventions. Read actual code, do not guess.

Do **not** change any files in phases 1–3.

**Done when:** you know which parts of the code are affected and can map out the decisions that need to be made.

---

## Phase 2 – Grill

Your job is to make the plan solid before any code is written. Be critical. A plan with unresolved gaps leads to code that does the wrong thing.

### The design tree

Treat the change as a **design tree**: every decision branches into decisions that depend on it. The **frontier** is the set of open decisions whose prerequisites are already settled. You can only ask questions from the frontier.

### Rounds

- Each round asks the **whole current frontier** at once.
- Questions that depend on each other never go into the same round. A question that depends on an open answer waits for a later round.
- After the user answers, the frontier moves outward. The next round asks what those answers unblocked.
- If a round would get very long (more than ~7 questions), ask the questions with the most dependent decisions first.

### Question format

Number every question, give it a short title, then the question, then your recommended answer on its own line:

```
### Round 2

❓ 1. Duplicate submissions
If the same order is submitted twice within a few seconds, should the second one be rejected?
➡️ Recommendation: No deduplication. CheckoutService has none either, and the client already disables the button.

❓ 2. Cancelled orders in the report
Should cancelled orders appear in the monthly report?
➡️ Recommendation: Yes, as a separate column, so totals stay comparable with the current report.
```

The user can answer by number ("1 yes, 2 no, because …"). Phrase each recommendation so that agreeing with it is unambiguous. If the recommendation contradicts the question as worded, say so explicitly.

### Facts vs. decisions

- **Facts are your job.** If a question can be answered by reading the code, config, build files, or git history, find out yourself instead of asking. State the fact briefly where it matters ("`Order.status` is persisted as a string, so a new enum value needs no migration").
- **Decisions belong to the user.** Never answer your own decision questions and move on. Wait for the answer.

### What to challenge

Look for gaps in these areas and ask about the ones that are relevant:

- **Scope:** what is explicitly *not* part of this change?
- **Behavior:** happy path, every edge case, empty/null/boundary input, duplicates, ordering
- **Error handling:** what happens on invalid input, missing data, failing collaborators; which exceptions, which error responses
- **Contradictions:** with existing code, with existing behavior, with earlier answers
- **Compatibility:** public APIs, DTOs, events/messages, database schema; are consumers affected; is a migration needed
- **Data & consistency:** transactions, concurrency, idempotency
- **Placement:** which class/module the logic belongs in, new class vs. extending an existing one
- **Operations:** configuration, feature flags, logging, performance on realistic data volumes
- **Security:** permissions, validation of external input

Push back on vague answers ("it should just work", "like usual"): ask what that means concretely. If an answer creates a new contradiction, raise it in the next round.

### When grilling ends

- The frontier is empty: every decision needed to write the code is settled.
- Or the user explicitly asks to wrap up. Then list the still-open points together with the default you would use, clearly marked, and let the user confirm them as part of the plan.

There is no question limit. Do not stop early just because there have been many questions.

**Done when:** no open decisions remain (or the user wrapped up and the open points are listed).

---

## Phase 3 – Plan

Present the plan **in the chat** (no file) with these sections:

1. **Goal** – one or two sentences
2. **Out of scope** – what is deliberately not done
3. **Decisions** – every decision from the grilling, one line each
4. **Changes** – per class/file: new or changed, and what changes; new or changed public method signatures written out
5. **Data & compatibility** – schema/migration, API/event changes, or "none"
6. **Error handling** – which cases, which behavior
7. **Order of implementation** – the steps
8. **Risks / open points** – or "none"

Tests are **not** part of this plan. They are written later with the `define-tests` and `implement-tests` steps.

Then ask for approval. Only an explicit approval counts ("ok", "passt", "go", "freigegeben"). If the user requests changes, update the plan, show the changed sections, and ask again.

**Done when:** the user has explicitly approved the plan.

---

## Phase 4 – Implement

### Write the code

- Implement **exactly the approved plan**. Follow the order of implementation.
- Follow the existing conventions of the repository (structure, naming, error handling, logging, dependency injection). Kotlin idioms over Java-style code.
- No unrelated refactorings, no "while I'm here" changes. Mention noticed improvements in the report instead.
- **If something comes up that the plan does not cover** or the plan turns out to be wrong (a hidden dependency, a contradiction in the code): stop, explain it, and ask in the round format. Do not decide it silently. Purely technical details with one obvious solution do not need a question.
- Do not write new tests (that is the job of `define-tests` / `implement-tests`).

### Verify

1. Detect the build tool (`gradlew` or `mvnw`) and compile production and test code of the affected modules, e.g. `./gradlew :<module>:compileKotlin :<module>:compileTestKotlin`.
2. If existing tests no longer compile because of a **planned** signature change, adapt them minimally so they compile (e.g. add a new constructor parameter). Do not change their assertions.
3. Run the existing tests of the affected modules. If tests fail because of **intended** behavior changes, do not adjust their assertions yourself: list them in the report. Fix failures caused by mistakes in your implementation.
4. If ktlint or detekt is configured, run it for the affected modules and fix findings in your code.

### Do not commit

Do not run `git add`, `git commit`, or `git push`. The user reviews the changes and commits themselves.

### Final report

Briefly output:
- changed and new files, one line each on what changed
- deviations from the plan (should be none, otherwise why)
- existing tests that fail because of intended behavior changes
- noticed improvements that were deliberately not done
- next step: `@define-tests`

### Commit message

End with a short commit message **in German**, in a code block so it can be copied:

1. Check the recent history (`git log --oneline -20`) and follow its conventions (e.g. ticket prefix, Conventional Commits, tense).
2. If the branch name contains a ticket number (e.g. `feature/ABC-123-…`) and the history uses ticket prefixes, include it.
3. Without a recognizable convention:
   - subject line in the imperative, at most 72 characters (e.g. `Stornierung von Aufträgen ermöglichen`)
   - optionally a blank line and 1–3 short bullet points on the most important changes

```
Stornierung von Aufträgen ermöglichen

- OrderService.cancelOrder setzt Status auf CANCELLED
- Versendete Aufträge können nicht storniert werden
```