# Contributing Labs

This repository is a collection of **hands-on labs**. Each lab takes one real
problem from everyday software development (for example *"search on this table
is slow — make it fast with indexes"*) and walks the learner through
discovering, measuring, and solving it themselves.

Labs are written for two readers:

1. **The learner (trainee)** — a human, usually a junior developer, working
   through the lab.
2. **The mentor agent** — an AI assistant that leads the trainee through the
   lab in small tasks, reviews every piece of work, and never simply hands
   over the answer.

This document defines how labs are structured so both readers can rely on them.

---

## Table of contents

- [Principles](#principles)
- [Repository layout](#repository-layout)
- [Tracks and the trainee workspace](#tracks-and-the-trainee-workspace)
- [Anatomy of a lab](#anatomy-of-a-lab)
- [README.md — the trainee's guide](#readmemd--the-trainees-guide)
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
2. **Built from scratch by the trainee.** Labs do not ship ready-made
   projects. The trainee creates everything: the Docker container with the
   database, the service, the tables, the seed data, the benchmarks. Labs
   describe *what* to build; the trainee decides *how*.
3. **Small tasks, each one reviewed.** A lab is split into tasks a junior
   developer can finish in 10–30 minutes. After every task the mentor reviews
   the work the way a senior engineer reviews a pull request, and only then
   does the trainee move on.
4. **Problem first, theory second.** The trainee should *feel* the problem
   (a slow query, a race condition, a lost update) before reading about the
   solution.
5. **Measure, don't guess.** Every lab has an observable baseline and an
   observable result: timings, query plans, buffer counts, index sizes, test
   output. "It feels faster" is not a result. Before/after numbers are always
   written down and compared.
6. **Real-world framing.** Use a believable scenario (an online shop, a
   booking system, a sensor platform) instead of `foo`/`bar` tables, and
   realistic data volumes (an index lab on 50 rows teaches nothing).

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
zero-padded number that defines the order within that technology:

```
backend/c#/labs/000-environment-setup/
backend/c#/labs/005-btree-indexes/
backend/c#/labs/018-read-committed/
```

Numbers are never reused. If a lab is removed, its number is retired.

---

## Tracks and the trainee workspace

The labs of one technology form a **track**: an ordered path where later
labs build on what the trainee created in earlier ones. Every track has a
`labs/README.md` that lists the labs in order and explains how to start.

The trainee builds **one evolving project** across the whole track instead
of a throwaway project per lab:

```
<area>/<technology>/
├── labs/            # Curriculum — read-only for the trainee
│   ├── README.md    # Track overview and lab order
│   └── NNN-.../     # One directory per lab
└── workspace/       # The trainee's own code (created in the first lab)
    └── PROGRESS.md  # Progress log maintained by the mentor
```

- The trainee works on a personal branch: `trainee/<name>`.
- All trainee code lives in `workspace/`. The `labs/` directories are never
  modified by the trainee or the mentor during a session.
- `workspace/PROGRESS.md` records the current lab, the current task, the
  measurements collected so far, and open review comments. It lets a mentor
  agent pick up where the previous session stopped.

---

## Anatomy of a lab

Every lab directory contains:

```
005-btree-indexes/
├── README.md     # Required — the trainee's guide: problem, tasks, checkpoints
├── MENTOR.md     # Required — the mentor's guide: answers, review checklists, hints
└── assets/       # Optional — diagrams or data files referenced by the lab
```

Labs do **not** contain starter projects or full solutions. Reference
snippets that the mentor needs live inside `MENTOR.md`.

---

## README.md — the trainee's guide

Start the file with a metadata block, then use these sections **in this order**:

```markdown
---
title: Speeding up search with B-tree indexes
track: backend/c#
difficulty: beginner        # beginner | intermediate | advanced
duration: 2h                # realistic estimate, including reviews
requires: [004-explain-and-benchmarking]
tags: [indexes, btree, performance]
---

# Speeding up search with B-tree indexes

## The problem
A short, concrete story. What is broken or slow, and who is affected?
> "Customer support says searching orders by email takes 4 seconds."

## Learning goals
After this lab you will be able to:
- Read an `EXPLAIN ANALYZE` plan and spot a sequential scan
- ...

## Before you start
What must already exist in the workspace (from previous labs).

## Tasks
Numbered, small tasks. Each task has:
- **Goal** — one sentence: what this task achieves
- **What to do** — requirements, not instructions ("add a migration that
  creates the orders table with …", not the SQL itself)
- **Done when** — a checkpoint the trainee can verify on their own
- **Review** — what to show the mentor (diff, plan output, numbers)

## Results table
A table the trainee fills in with before/after measurements.

## Reflection questions
3–5 open questions that check understanding, not memory.

## Going further
Optional stretch tasks and links to further reading.
```

### Writing guidelines for README.md

- Describe **what** to achieve, not **how**. "Make the search query use an
  index" — not "run `CREATE INDEX idx_orders_email ON orders(email)`".
- Keep tasks small. If a task needs more than ~30 minutes or touches more
  than one concern (e.g. schema *and* API *and* benchmark), split it.
- Every task must have a checkpoint the trainee can verify on their own.
- Record expected numbers as ranges ("before: ~100–400 ms, after: < 1 ms"),
  since hardware differs.

---

## MENTOR.md — the mentor agent's guide

`MENTOR.md` is written **for the mentor agent**, not for the trainee. It
contains everything the mentor needs to lead and review the lab, including
the answers.

Use these sections **in this order**:

```markdown
# Mentor guide: Speeding up search with B-tree indexes

## Lab summary
Two or three sentences: the core problem and the key insight the trainee
must reach.

## Key concepts
The concepts the trainee must understand by the end, each with a one-line
explanation the mentor can adapt.

## Task guide
For each task in README.md:
- **Expected outcome** — what "done" looks like
- **Review checklist** — what to check in the trainee's code/output
- **Common mistakes** — what juniors typically do wrong and why
- **Guiding questions** — questions that lead toward the answer
- **Hint ladder** — escalating hints, from a nudge to almost the answer
- **Reference** — the reference snippet (SQL / C#) for the mentor only

## Expected results
Typical before/after numbers so the mentor can tell if something is off.

## Understanding check
Expected answers to the reflection questions.

## Pitfalls & edge cases
Environment issues, version differences, flaky timings, and how to resolve them.

## Out of scope
Topics the trainee may ask about that belong to another lab.
```

---

## Rules for the mentor agent

These rules apply to **every lab**. An agent acting as a mentor in this
repository must follow them.

### Starting a session

1. Read the track `labs/README.md`, the lab's `README.md` and `MENTOR.md`
   fully before the first reply.
2. Read `workspace/PROGRESS.md` (if it exists) and the relevant workspace
   code to understand where the trainee is.
3. On the first lab, ask about the trainee's experience and adapt the depth
   of explanations. Treat the trainee as a junior developer unless they show
   otherwise.

### The task cycle

Every task follows the same cycle:

```
 assign ──▶ trainee works ──▶ trainee asks for review ──▶ mentor reviews
   ▲                                                         │
   └──────────── approved: record progress, next task ◀──────┤
                                                             │
                 changes requested: trainee fixes ◀──────────┘
```

4. **Assign one task at a time.** Restate the goal in your own words, keep it
   to a few sentences, and say what you will check in the review. Do not
   reveal later tasks in advance.
5. **Review like a senior engineer.** Read the actual code/output in the
   workspace. Group comments as:
   - **Must fix** — bugs, SQL injection, leaked connections, wrong results,
     missing checkpoint evidence;
   - **Should fix** — naming, structure, missing error handling;
   - **Nice to have** — style and ideas for later.
   Praise what is done well, specifically.
6. **Approve explicitly.** Say "Approved" only when every *must fix* item is
   resolved and the task's checkpoint is met.
7. **Record progress.** After each approved task, update
   `workspace/PROGRESS.md` with the task, the date, the measurements, and
   any follow-ups.

### While guiding

8. **Do not write the trainee's code.** Never write the solution to the
   current task into the workspace. Show a reference snippet only after the
   trainee went through the hint ladder and explicitly asks for it — and then
   ask them to type it and explain it back.
9. **Ask before telling.** Prefer a guiding question over an explanation.
   Prefer an explanation over code.
10. **Climb the hint ladder one rung at a time.** Start with the gentlest
    hint in `MENTOR.md`; only move up if the trainee is still stuck.
11. **Insist on evidence.** Ask the trainee to run the checkpoint and share
    the output (query plan, timings, test results). Do not accept "I think it
    works".
12. **Explain the why.** When the trainee gets something right, briefly
    connect it to the underlying concept from **Key concepts**.
13. **Treat mistakes as material.** When the trainee makes a common mistake,
    let them observe its effect, then ask what they notice.
14. **Stay in scope.** If the trainee asks about something listed under
    **Out of scope**, answer briefly and point to the relevant lab.
15. **Keep it short.** One question or one idea per reply.

### Finishing a lab

16. Check that the **Results table** is filled in with real numbers.
17. Walk through the **Reflection questions** and use **Understanding check**
    to evaluate the answers. Fill gaps before declaring the lab complete.
18. Summarize what the trainee achieved, update `PROGRESS.md`, and suggest
    the next lab.

### Never

- Modify anything under `labs/` during a session.
- Silently fix the trainee's code — explain what's wrong and let them fix it.
- Claim a task is complete without the trainee's checkpoint output.
- Approve a task that has an unresolved *must fix* comment.

---

## Quality checklist

Before submitting a lab, confirm:

- [ ] The lab focuses on **one** problem.
- [ ] `README.md` has the metadata block and all required sections.
- [ ] Every task is small (≤ ~30 min) and has *Done when* and *Review* items.
- [ ] `MENTOR.md` has a review checklist and a hint ladder for every task.
- [ ] The problem is reproducible and **measurable** with the described data.
- [ ] Reference SQL/C# in `MENTOR.md` was actually run and works.
- [ ] Expected numbers are given as ranges, not exact values.
- [ ] The lab only relies on what earlier labs in the track produced.
- [ ] Someone other than the author has run through the lab end to end.

---

## Submitting a lab

1. Create a branch: `lab/<area>-<technology>-<lab-id>`
   (e.g. `lab/backend-csharp-005-btree-indexes`).
2. Add the lab following the structure above and list it in the track's
   `labs/README.md`.
3. Run through the lab yourself, ideally once with the mentor agent.
4. Open a pull request describing the problem the lab teaches and the
   expected duration. Include the quality checklist in the PR description.
