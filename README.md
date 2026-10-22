# verbose-sandbox

A collection of **hands-on labs** for junior developers, guided by an **AI
mentor**.

Each lab takes one real problem from everyday development, like *"searching
orders is slow"*. The trainee builds everything from scratch, measures the
problem, and fixes it. They don't read about the solution first.
An AI agent acts as a mentor. It assigns small tasks one at a time, reviews
the work like a senior engineer, and gives hints instead of answers.

## How it works

```
 assign task ──▶ trainee builds ──▶ mentor reviews ──▶ approved? ──▶ next task
                      ▲                                    │ no
                      └────────────── fix comments ◀───────┘
```

- **Problem first, theory second.** You feel the slow query before you learn
  about indexes.
- **Measure, don't guess.** Every lab has before/after numbers, such as
  timings, query plans, and sizes.
- **You write the code.** Labs describe *what* to build, not *how*. The
  mentor never writes your solution.
- **One project per track.** Across a track you grow a single project, and
  each lab builds on the previous ones.

## Tracks

| Track | Status | Labs |
|-------|--------|------|
| [`backend/c#`](backend/c%23/labs/README.md) | Available | [001 — Speeding up search with B-tree indexes](backend/c%23/labs/001-btree-indexes/README.md) |
| `backend/python` | Planned | 001 — Algorithms |
| `database/postgresql` | Planned | — |
| `frontend/angular` | Planned | — |

## Getting started (trainee)

1. Install the tools listed in your track's `labs/README.md`. For the C#
   track, you need the .NET SDK 9+ and Docker.
2. Install [Claude Code](https://claude.com/claude-code) and open this
   repository in it.
3. Create your own branch:
   ```bash
   git switch -c trainee/<your-name>
   ```
4. Start the mentor:
   ```
   /mentor backend/c#/labs/001-btree-indexes
   ```
   Later, run just `/mentor` to continue where you stopped.

Your code goes into `<area>/<technology>/workspace/`. The mentor tracks your
progress in `workspace/PROGRESS.md`.

## Repository layout

```
<area>/<technology>/
├── labs/                    # Curriculum, read-only for trainees
│   ├── README.md            # Track overview and lab order
│   └── NNN-lab-name/
│       ├── README.md        # Trainee's guide: problem, tasks, checkpoints
│       └── MENTOR.md        # Mentor's guide: review checklists, hints, answers
└── workspace/               # Trainee's code (created in the first lab)

.claude/skills/mentor/       # The /mentor skill that runs mentoring sessions
CONTRIBUTING.md              # How to write labs, and the rules for the mentor
```

`MENTOR.md` files contain the answers. As a trainee, you'll learn more if you
don't open them.

## Contributing

Want to add a lab? Read [CONTRIBUTING.md](CONTRIBUTING.md). It covers the
lab structure, the required sections, the mentor rules, and the quality
checklist.
