---
name: opencode-first
description: "Orchestrate: Claude plans/reviews, fans out implementation to parallel opencode workers (Grok 4.5). Plan big, execute small."
---

# opencode First

Claude Code sessions only. Other harnesses: skip; never self-delegate.

Rationale: Claude (Fable/Opus) tokens metered + expensive; Grok 4.5 via opencode is the flat-rate/cheaper worker. Grok is fast at writing/implementing code from a clear spec; Claude wins at ergonomics — judgment, design, spec-writing, review, orchestration. So opencode types, Claude thinks and verifies.

Model: Claude is the **orchestrator** (plan big); opencode workers **execute small**. The win is keeping generation + exploration tokens on the cheaper worker while Claude spends only on aligning, planning, the fan-out specs, and review (diffs + cross-worker synthesis). Mirror the "plan big, execute small" coordinator pattern — Claude never does worker-level *bulk* typing/reading when a worker can.

Worker engine: `opencode run --model xai/grok-4.5 --variant high` (v1.17+; `--auto` auto-approves, `--variant high` = high reasoning effort). **Always pass `--variant high`** — a headless `run` does not inherit the effort picked in the opencode TUI; omitting `--variant` uses grok-4.5's default variant.

## Route

Delegate to opencode (default for hands-on work):

- implementation from a frozen spec; refactors; mechanical migrations
- bug fixes with known repro; test writing; coverage fills
- CI fixes, dependency bumps, scripts/tooling
- bulk codebase exploration where raw reading ≫ the answer
- **all web search / web research** — always delegate to Grok (it has web fetch); never use Claude's own WebSearch/WebFetch

Keep in Claude:

- design, API design, architecture, naming, UX judgment
- tasks where writing the spec IS the work (ambiguity = design)
- tiny edits (~<20 lines, single obvious change) — delegation overhead loses
- anything needing session tools: MCP (browser/computer-use/chronicle), 1Password, secrets
- destructive/irreversible ops, releases, pushes, GitHub mutations — Claude-side per git rules
- review of worker output — never delegated, never skipped
- **decomposition + synthesis** — splitting the plan and merging worker results is orchestrator work

Mixed task: Claude designs first, freezes spec, delegates build-out.
Heuristic: prompt reads as a work order → delegate; writing it forces decisions → design, Claude.

## Align (grill) first — before any fan-out

Workers start with zero context and only their frozen spec, so **spec quality decides success** — and a vague spec fails silently across N workers at once. Before decomposing, grill the user until intent is locked, then freeze it as `PLAN.md`. That file is the single source the fan-out specs are cut from.

When to grill: any non-trivial or ambiguous task. Skip for a tiny obvious edit or an already-fully-specified work order.

Grill loop:
- surface open decisions with `AskUserQuestion` — batch related ones (up to 4 per call), **recommended option first**, so the user picks defaults fast instead of a slow ping-pong
- resolve facts by the cheapest source, in order: (1) **cheap targeted reads** Claude does directly (one file, a config, a signature); (2) **bulk exploration** → delegate a Grok scout (Route → bulk codebase exploration) and fold its distilled findings into the plan — never read the whole codebase into Claude's context; (3) ask the user only what neither can settle (intent, priorities, tradeoffs)
- walk the whole decision tree — resolve dependencies, don't stop at the first unknown
- stop when nothing material is unresolved; don't grill past alignment

Then write `PLAN.md` — a spec doc, not raw file dumps (low-volume Claude output; the scouting behind it is delegated, the judgment that writes it is Claude's). It's the frozen spec — nothing gets *built* before it exists, but its inputs may themselves be delegated:

```
# Plan: <task>
## Goal
## Approach
## Key decisions & tradeoffs
## Risks / open questions
## Out of scope
```

No code is written during grilling. Decomposition (below) cuts each worker's spec from `PLAN.md`'s Goal/Approach/decisions/out-of-scope — so constraints and non-goals propagate to every worker automatically.

## Orchestrate (fan out)

When a task splits into independent sub-tasks (per-file, per-module, per-park), fan out — don't run them serially. Each worker gets its own frozen spec + its own output file + its own background job.

Decomposition rules:
- subtasks must be **genuinely independent** — no worker needs another's output mid-run; shared edits to the same file → keep serial or single-worker
- right-size the brief: each opencode session carries fixed setup overhead — too fine raises cost, too coarse loses parallelism. Aim for a handful of substantial workers, not dozens of trivial ones
- Claude decomposes, launches, then **waits** — it does not start implementing subtask 1 itself while workers run

Fan-out loop:

```bash
# one worker per independent subtask — launch each as a SEPARATE Bash
# run_in_background call (N = 1, 2, 3…), each with its own spec + output file.
# Quoted heredoc (<<'EOF') so $ / $(...) / backticks in the spec are literal, not executed.
P=$(mktemp); cat >"$P" <<'EOF'
<goal for THIS subtask, repo + key paths, constraints ("don't touch X"), non-goals, proof expected, output shape>
EOF
opencode run --model xai/grok-4.5 --variant high --auto --dir <repo> "$(cat "$P")" \
  > /tmp/opencode-worker-N.md 2>/dev/null </dev/null
```

- launch each worker with Bash `run_in_background: true` (separate calls) for long/parallel runs; read its `/tmp/opencode-worker-N.md` on exit
- **`</dev/null` is load-bearing**: under a harness background shell, `opencode run` can block forever on the never-closing stdin pipe — alive at ~0% CPU, zero tool calls, nothing logged past bootstrap, output file empty. Short prompts sometimes survive, so it looks intermittent. Always redirect stdin on background runs; foreground + `timeout N` also works for runs under ~10 min
- **watchdog every background fan-out**: `run` prints only at completion, so a wedged worker is indistinguishable from a working one. Arm one background loop that exits loudly on: first files landing (touch-marker + `find -newer`), all workers exited, or N minutes with no writes (12–15 min). A stall alarm means kill + relaunch, not wait
- keep each worker scoped to its subtask's paths — scope is efficiency *and* safety (a cheap worker can't wander)
- collect all output files, then synthesize — only Claude sees the full picture across workers

## Research (web) — always Grok

**Never use Claude's own WebSearch/WebFetch.** All web lookups — quick single
facts included — go to a Grok worker, which has a web fetch tool. Workers read
pages in their own context and return **distilled findings**, not raw pages.

Why quick web facts delegate but quick codebase reads don't: a one-file read is cheap and local, while web pages are token-heavy raw content that would bloat Claude's context — so all web, even a single fact, goes to Grok.

Single lookup:

```bash
P=$(mktemp); cat >"$P" <<'EOF'
Research question: <question>. Use your web fetch tool. Cross-check across
sources. Return: the answer + the source URLs + a one-line confidence note.
EOF
opencode run --model xai/grok-4.5 --variant high --auto "$(cat "$P")" > /tmp/opencode-research.md 2>/dev/null </dev/null
```

Research with multiple web fetches can legitimately run 5–15 min; give
background research runs an internal ceiling (`timeout 900 opencode run …`)
rather than assuming a quiet run has hung.

Multi-part research: decompose into independent sub-questions and **fan out**
one worker per sub-question (see Orchestrate) — each returns distilled findings
to its own output file; Claude synthesizes. `--dir` optional for research (no
repo needed); add `--dir <repo>` only when the question is about local code.

## Invoke (single)

Prompt via temp file, never inline quoting; pass the file's contents as the run message:

```bash
P=$(mktemp); cat >"$P" <<'EOF'
<goal, repo + key paths, constraints ("don't touch X"), non-goals, proof expected, output shape>
EOF
opencode run --model xai/grok-4.5 --variant high --auto --dir <repo> "$(cat "$P")" \
  > /tmp/opencode-last.md 2>/dev/null </dev/null
```

- `--auto` auto-approves permissions not explicitly denied (the house default); keep prompts scoped to the target repo
- `--model xai/grok-4.5` pins the worker; confirm ids with `opencode models | grep grok`
- `--variant high` sets grok's reasoning effort (values: `high`/`max`/`minimal`). **Required** — without it the run uses the default variant, not the effort picked in the TUI
- `--dir <repo>` sets the working directory
- opencode `run` prints the result to stdout — redirect with `> file`; stderr suppressed (noise bloats context), drop `2>/dev/null` only to debug a failing run
- long runs: Bash run_in_background **with `</dev/null`** (see Orchestrate — omitting it can wedge the run on the background shell's stdin pipe), read the output file on exit; don't kill quiet runs <30 min *unless* a watchdog's no-writes alarm fired
- debugging a wedged/failing run: rerun foreground with `timeout N` and `--print-logs --log-level DEBUG` — shows exactly where it stops (bootstrap, stream, tool dispatch)
- parallel independent tasks OK: separate repos/dirs, separate output files
- cold-boot overhead adds up across many workers: `opencode serve` once, then `opencode run --attach http://localhost:4096 ...` to reuse the server

Follow-up fixes — cheaper than fresh runs, keeps context:

```bash
# resume the most recent session in this repo
opencode run -c --variant high --auto --dir <repo> "$(cat "$P2")" > /tmp/opencode-last.md 2>/dev/null
```

For a specific parallel worker, resume by id: run the initial worker with `--format json` (or `opencode sessions`) to capture its session id, then `opencode run --session <id> --variant high --auto ...`.

## Prompt contract

The worker starts with zero session context. Every prompt: goal, exact repo/paths, constraints, non-goals, proof expected (exact test command), output shape ("report files changed + test output"). Spec quality decides success.

## Verify (Claude, always)

- `git status -sb` + read the full diff; judge like a contributor PR
- run focused tests yourself or demand proof output; worker claims are advisory
- iterate via resume; after 2 failed rounds, take over and do it directly
- normal closeout still applies: `$autoreview` before ship

## Economics

Win = generation + exploration tokens moved to opencode/Grok, parallelized across workers; Claude spends only on aligning + planning, fan-out specs, and review (diffs + synthesis). Don't ping-pong trivia through delegation; don't re-read what a worker already summarized. If a run ends with nothing delegated, you paid a frontier round-trip for no benefit — decompose or hand it to a worker.
