---
name: smart-task-router
description: Use when the user asks to build, fix or run any task, from a small edit to a large multi-file project. Plans with a strong model, picks the cheapest model and effort per subtask, verifies every step, escalates only on failure, and keeps state so work can resume.
---

# Smart Task Router

Principle: **strong model thinks, cheap models type, real checks decide.** Opus plans and reviews. Sonnet or Haiku build small, well-specified pieces. Every piece is proven by a check before it counts.

## 0. Pick the mode

- **Answer:** a question or explanation. Reply directly, with no agents.
- **Small task:** one concern, about 3 files or fewer, clear result. Use the Quick path.
- **Project:** a feature across modules, a new app or service, a migration, an integration, or anything vague or large. Use the Project path.
- **"Continue" / "resume":** read `.router/PROGRESS.md` and pick up at the first unfinished task.

## Model tiers (used by both paths)

| Tier | Agent | Model / effort | Typical work |
|---|---|---|---|
| E Easy | `router-easy` | haiku, low | Boilerplate, renames, formatting, data conversion, tests written from a given spec, copying an existing pattern, docs |
| N Normal | `router-normal` | sonnet, medium | Standard features inside defined interfaces, UI components, scripts, endpoints that follow existing patterns |
| H Hard | `router-hard` | opus, high | Planning, core logic, interfaces and contracts, schemas, unclear bugs, security, production data, final review |
| X Last resort | none (model override) | fable, high | Only after H fails twice, or when the user asks |

Raise the tier for: a vague spec, no way to check the result, irreversible actions, many unknown files, or an earlier failure.
Lower the tier for: clear examples, existing tests, a repeated pattern, or a reversible change.

**Spawning:** if the plugin's agents are installed, use them by name (they may appear namespaced, e.g. `smart-task-router:router-easy`), because they also set effort. Otherwise spawn a general agent with the tier's model as an override. If no agent tool exists at all, see "Without an agent tool" below.

---

## Quick path (small tasks)

1. Write a spec: goal, files, exact output, constraints, done-check.
2. Delegate it to the tier's agent (or do it yourself if spawning would cost more than the edit).
3. Run the done-check yourself.
4. On failure, re-run one tier up and add the error to the spec. Escalate at most twice.
5. Report in one line: `E -> failed -> N -> passed`.

---

## Project path (big tasks)

All state lives in `.router/` in the project root. It lets cheap agents work without the whole conversation, and it lets work resume in a new session. Suggest adding `.router/` to `.gitignore` unless the user wants to keep it.

### Phase 1: Recon (cheap, read-only)

Spawn `router-scout` (haiku) to write `.router/CONTEXT.md`, at most about 150 lines:

- Stack, frameworks and versions, plus exact run, build, test and lint commands.
- Folder map of relevant areas, entry points and config files.
- Conventions: naming, error handling, state management, styling, and an example file to copy patterns from.
- Files likely to be touched, and external services (APIs, DB, deploy target).

The codebase is explored **once**. Every later agent reads CONTEXT.md instead of re-exploring.

### Phase 2: Plan (Tier H)

Write `.router/PLAN.md` with:

1. **Goal, non-goals and acceptance criteria:** testable statements of "done".
2. **Decisions:** architecture choices and their reasons.
3. **Contracts first:** function signatures, API request and response shapes, DB schema, shared types, env vars. Contracts let cheap agents build pieces in parallel without guessing.
4. **Task list.** Each task has:
   - `id`, `title`, `tier`, `depends_on`
   - `files`: exact paths to create or edit (one concern, about 3 files or fewer per task)
   - `spec`: what to build, which contract it implements, and which pattern to copy
   - `done_check`: a command or concrete observation

Order the tasks as contracts and scaffolding, then core logic, then features, then UI, then tests and docs.
**If the user is present, show the plan summary and get approval before building.** Changing a plan is cheap. Rebuilding is expensive.

### Phase 3: Build (task by task)

For each ready task (all of its dependencies done):

1. Spawn the task's tier agent. Its prompt contains only:
   - the path to CONTEXT.md ("read this first")
   - the relevant contracts copied from PLAN.md
   - the task spec, the file list and the done-check
   - the rule: "Edit only the listed files. If something outside them must change, stop and report it."
2. Tasks with **no file overlap** can run in parallel (use worktree isolation if available).
3. Run the done-check yourself. Never trust the agent's report alone.
4. Pass: mark it done in `.router/PROGRESS.md` (task id, tier used, files changed, one-line note). Commit if git is used, one commit per task as a checkpoint.
5. Fail: escalate one tier and include the error output plus what was tried.
6. **Failed twice at Tier H means the plan is wrong for this part.** Return to Phase 2, re-plan that part (split it, fix the contract) and continue. Never brute-force.

Agent reports must be short: pass or fail, files changed, any deviation from the spec, and open issues. No full file dumps.

### Phase 4: Integrate and review (Tier H)

1. Run the full build, tests and lint, then start the app and exercise the main flow.
2. `router-reviewer` (read-only) checks the **diff**, not the whole repo, against the acceptance criteria, the contracts, security and conventions.
3. Turn each finding into a new task in PLAN.md and route it through Phase 3 at the right tier.
4. Repeat until the acceptance criteria pass.

### Phase 5: Report

Report in a short summary: what was built, how it was verified, the tier mix (e.g. `12 tasks: 5 E, 5 N, 2 H, 1 escalation`), and anything left for the user (keys to add, deploy steps, decisions).

---

## Token rules

- Pass **paths, not file contents**. Agents read only the files listed for them.
- Do not re-explore what CONTEXT.md already covers. Update CONTEXT.md when you learn something new.
- One concern per task. Big vague tasks cost more and fail more.
- When the conversation grows long, rely on the `.router/` files, not on recalling earlier messages.
- If escalations exceed about 30% of tasks, stop and fix the plan or specs. The specs are too thin.
- Do not route trivial edits through agents, because spawning one costs more than doing it.

## Safety

- Ask the user before irreversible actions: deploys, migrations on production databases, changes to live business data, deleting files, or sending messages.
- Secrets go in env vars and are never written into code or `.router/` files.

## Without an agent tool (plain chat)

Follow the same phases yourself in order: recon notes, plan with contracts, build task by task, check each one, then review. Tell the user in one line which model and effort would suit the build phase next time.
