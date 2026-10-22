# Contributing Labs

This repository is a collection of **hands-on labs**. Each lab takes one real
problem from everyday software development (for example *"search on this table
is slow — make it fast with indexes"*) and walks the learner through
discovering, measuring, and solving it themselves.

Labs are written for two readers:

1. **The learner** — a human working through the lab.
2. **The mentor agent** — an AI assistant that guides the learner through the
   lab without simply handing over the answer.

This document defines how labs are structured so both readers can rely on them.

---

## Table of contents

- [Principles](#principles)
- [Repository layout](#repository-layout)
- [Anatomy of a lab](#anatomy-of-a-lab)
- [README.md — the learner's guide](#readmemd--the-learners-guide)
- [MENTOR.md — the mentor agent's guide](#mentormd--the-mentor-agents-guide)
- [Rules for the mentor agent](#rules-for-the-mentor-agent)
- [Quality checklist](#quality-checklist)
- [Submitting a lab](#submitting-a-lab)

---

## Principles

Every lab must follow these principles:

1. **One problem per lab.** A lab answers a single question, such as "Why is
   this query slow and how do indexes fix it?". If you find yourself writing
   "and also…", split it into two labs.
2. **Problem first, theory second.** The learner should *feel* the problem
   (a slow query, a race condition, a memory leak) before reading about the
   solution.
3. **Measure, don't guess.** Every lab has an observable baseline and an
   observable result: timings, query plans, test output, memory graphs. "It
   feels faster" is not a result.
4. **Runnable in minutes.** Setup should take one or two commands. Prefer
   Docker / docker compose for anything that needs a database or service.
5. **Learner does the work.** Starter code is incomplete on purpose. The
   reference solution exists for the mentor and for self-checking, not for
   copying.
6. **Real-world framing.** Use a believable scenario (an online shop, a
   ticketing system, a log store) instead of `foo`/`bar` tables.

---

## Repository layout

Labs are grouped by **area** and **technology**:

```
<area>/<technology>/labs/<lab-id>/
```

| Area       | Examples of technology          |
|------------|---------------------------------|
| `backend`  | `python`, `c#`                  |
| `database` | `postgresql`                    |
| `frontend` | `angular`                       |

Shared, cross-lab material (glossaries, diagrams, general guides) goes in
[`docs/`](docs/).

### Lab ID

Lab directories are named `NNN-short-kebab-name`, where `NNN` is a
zero-padded number that defines the recommended order within that
technology:

```
database/postgresql/labs/001-btree-indexes/
database/postgresql/labs/002-composite-indexes/
database/postgresql/labs/003-n-plus-one-queries/
```

Numbers are never reused. If a lab is removed, its number is retired.

---

## Anatomy of a lab

Every lab directory contains:

```
001-btree-indexes/
├── README.md          # Required — the learner's guide
├── MENTOR.md          # Required — the mentor agent's guide
├── docker-compose.yml # If the lab needs services (DB, cache, queue…)
├── setup/             # Seed data, migrations, scripts to reach the start state
├── starter/           # The code the learner starts from (intentionally incomplete)
├── solution/          # Reference solution — learners should not open it until done
└── checks/            # Automated checks / tests that verify each step
```

Only `README.md` and `MENTOR.md` are mandatory; include the rest when the lab
needs them.

---

## README.md — the learner's guide

Start the file with a metadata block, then use these sections **in this order**:

```markdown
---
title: Speeding up search with B-tree indexes
area: database
technology: postgresql
difficulty: beginner        # beginner | intermediate | advanced
duration: 45m               # realistic estimate
prerequisites:
  - Basic SQL (SELECT, WHERE, JOIN)
  - Docker installed
tags: [indexes, performance, query-planner]
---

# Speeding up search with B-tree indexes

## The problem
A short, concrete story. What is broken or slow, and who is affected?
> "Customer support says searching orders by email takes 4 seconds."

## Learning goals
After this lab you will be able to:
- Read an `EXPLAIN ANALYZE` plan and spot a sequential scan
- Create a B-tree index and verify the planner uses it
- Explain the write-cost trade-off of adding indexes

## Setup
Exact commands to reach the starting state, and how to confirm it worked.

## Steps
Numbered steps. Each step has:
- **Task** — what to do (not how to do it)
- **Checkpoint** — how the learner knows the step is done
  (expected output, command to run, number to compare)

## Hints
Collapsed hints, from gentle to specific, per step:
<details><summary>Hint 1 for step 2</summary>…</details>

## Reflection questions
3–5 open questions that check understanding, not memory.
> "Why didn't the planner use your index when the table had only 100 rows?"

## Going further
Optional stretch tasks and links to further reading.

## Cleanup
How to tear everything down.
```

### Writing guidelines for README.md

- Describe **what** to achieve, not **how**. "Make the search query use an
  index" — not "run `CREATE INDEX idx_orders_email ON orders(email)`".
- Every step must have a checkpoint the learner can verify on their own.
- Provide realistic data volume. Index labs are pointless on 50 rows —
  seed enough data that the problem is visible (e.g. 1M+ rows).
- Record expected numbers as ranges ("before: ~1–4 s, after: < 10 ms"),
  since hardware differs.

---

## MENTOR.md — the mentor agent's guide

`MENTOR.md` is written **for the AI mentor**, not for the learner. It contains
everything the mentor needs to guide the learner well, including the answers.

Use these sections **in this order**:

```markdown
# Mentor guide: Speeding up search with B-tree indexes

## Lab summary
Two or three sentences: the core problem and the key insight the learner
must reach.

## Key concepts
The concepts the learner must understand by the end, each with a one-line
explanation the mentor can adapt.

## Step-by-step guidance
For each step in README.md:
- **Expected outcome** — what "done" looks like
- **Common mistakes** — what learners typically do wrong and why
- **Guiding questions** — questions that lead toward the answer without giving it
- **Hint ladder** — escalating hints, from a nudge to almost the answer
- **How to verify** — commands/checks the mentor can ask the learner to run

## Reference solution
The full solution with an explanation of *why* it works.
(Or point to `solution/` and explain it.)

## Understanding check
How to tell if the learner truly understood vs. got lucky:
questions and expected answers for the reflection section.

## Pitfalls & edge cases
Environment issues, version differences, flaky timings, and how to resolve them.

## Out of scope
Topics the learner may ask about that belong to another lab.
Point to that lab instead of going deep here.
```

---

## Rules for the mentor agent

These rules apply to **every lab**. An agent acting as a mentor in this
repository must follow them.

### Before starting

1. Read the lab's `README.md` and `MENTOR.md` fully before the first reply.
2. Ask the learner about their experience with the prerequisites and adapt
   the depth of explanations accordingly.
3. Help the learner complete **Setup** and confirm the starting state works
   (e.g. the slow query is actually slow) before moving on.

### While guiding

4. **Do not give away the solution.** Never write the answer for the
   learner's current step unless they explicitly ask for it *after* having
   gone through the hint ladder.
5. **Ask before telling.** Prefer a guiding question over an explanation.
   Prefer an explanation over code.
6. **Climb the hint ladder one rung at a time.** Start with the gentlest hint
   in `MENTOR.md`; only move up if the learner is still stuck.
7. **One step at a time.** Do not jump ahead. Move to the next step only when
   the current step's checkpoint is met.
8. **Insist on evidence.** Ask the learner to run the checkpoint and share
   the output (query plan, timings, test results). Do not accept "I think it
   works".
9. **Explain the why.** When the learner gets something right, briefly
   connect it to the underlying concept from **Key concepts**.
10. **Treat mistakes as material.** When the learner makes a common mistake,
    let them observe its effect, then ask what they notice.
11. **Stay in scope.** If the learner asks about something listed under
    **Out of scope**, answer briefly and point to the relevant lab.
12. **Keep it short.** Mentor replies should be focused — one question or one
    idea at a time.

### Finishing

13. Walk through the **Reflection questions** and use **Understanding check**
    to evaluate the answers. Fill gaps before declaring the lab complete.
14. Summarize what the learner achieved: the before/after measurements and
    the concepts they now understand.
15. Suggest the next lab or a **Going further** task.

### Never

- Modify `solution/`, `checks/`, or `MENTOR.md` during a session.
- Silently fix the learner's code — explain what's wrong and let them fix it.
- Claim a step is complete without the learner's checkpoint output.

---

## Quality checklist

Before submitting a lab, confirm:

- [ ] The lab focuses on **one** problem.
- [ ] `README.md` has the metadata block and all required sections.
- [ ] `MENTOR.md` has all required sections, including a hint ladder for every step.
- [ ] Setup works from a clean clone in one or two commands.
- [ ] The problem is reproducible and **measurable** at the start.
- [ ] Every step has a checkpoint the learner can verify themselves.
- [ ] The reference solution passes all checks in `checks/`.
- [ ] Expected numbers are given as ranges, not exact values.
- [ ] Starter code does not accidentally contain the solution.
- [ ] Cleanup instructions remove everything the lab created.
- [ ] Someone other than the author has run through the lab end to end.

---

## Submitting a lab

1. Create a branch: `lab/<area>-<technology>-<lab-id>`
   (e.g. `lab/database-postgresql-001-btree-indexes`).
2. Add the lab following the structure above.
3. Run through the lab yourself, ideally once with the mentor agent.
4. Open a pull request describing the problem the lab teaches and the
   expected duration. Include the quality checklist in the PR description.
