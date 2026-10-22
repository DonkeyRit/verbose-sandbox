# C# backend track

Hands-on labs for C# backend developers. Across the track you build **one
evolving project** — a small online shop backend on .NET and PostgreSQL — and
use it to feel, measure, and fix real problems.

Read [CONTRIBUTING.md](../../../CONTRIBUTING.md) once to understand how labs,
the mentor agent, and the review cycle work.

## Labs

| #   | Lab                                                  | Difficulty | Duration | Requires |
|-----|------------------------------------------------------|------------|----------|----------|
| 001 | [Speeding up search with B-tree indexes](001-btree-indexes/README.md) | beginner | 4h | — |

## Before the first lab

You need:

- **.NET SDK 9** or newer — `dotnet --version`
- **Docker** with Docker Compose — `docker compose version`
- An editor or IDE for C# (Rider, Visual Studio, or VS Code with C# Dev Kit)
- Basic C# (classes, `async`/`await`) and basic SQL (`SELECT`, `JOIN`, `WHERE`)

## How to work through a lab

1. Create your personal branch: `git switch -c trainee/<your-name>`.
2. Open the repository in Claude Code and start the mentor:
   ```
   /mentor backend/c#/labs/001-btree-indexes
   ```
   Next time, just run `/mentor` — it continues from `workspace/PROGRESS.md`.
3. The mentor assigns one task at a time. Do the task in
   `backend/c#/workspace/`, then ask the mentor for a review.
4. Fix what the review asks for. When the task is **Approved**, the mentor
   records your progress in `workspace/PROGRESS.md` and you move on.

Everything you write lives in `backend/c#/workspace/`. Files under `labs/` are
the curriculum — don't change them.
