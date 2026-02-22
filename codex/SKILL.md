<!-- Location: ~/.claude/commands/codex.md -->
# /codex — Delegate tasks to OpenAI Codex CLI

## Delegation protocol

When this skill is invoked, the **main conversation** does only this:

1. Determine mode (implement / advise / review) from the user's intent.
2. Identify the project root (nearest `.git` or project marker). If no
   project root can be identified (no `.git`, no project markers), ask the
   user to specify the project directory. Do not default to the home directory.
3. **Implement mode only**: run `git status` to check for a dirty tree (this
   counts as a scouting call — max 2 scouting calls total before delegating).
4. Delegate the full execution sequence to a Task sub-agent (`model: "opus"`),
   passing this skill file as a Key File along with the resolved project root,
   selected mode, and the user's task description.

The main conversation does **not** run Codex commands inline. Everything below
is execution instructions for the sub-agent.

---

## Sub-agent execution instructions

### Resolve paths

At the start of execution, set `PROJECT_ROOT` to the project root passed by
main. All file paths below are relative to this variable.

### When NOT to use Codex

- **Tasks needing conversation context**: Codex starts fresh — it cannot
  see prior discussion, decisions, or context from this session.
- **Quick single-file edits**: Claude is faster; no sandbox overhead.
- **Raw data manipulation**: Risk of violating immutability rules. Keep
  raw data operations in Claude where safeguards are active.
- **Tasks requiring project-specific judgment**: When the value is Claude's
  understanding of project patterns, not raw coding throughput.

### Mode selection

| Intent                                  | Mode          | Command base                                        |
|-----------------------------------------|---------------|-----------------------------------------------------|
| Write or modify code                    | **implement** | `codex exec --sandbox workspace-write --full-auto`  |
| Get suggestions without changing files  | **advise**    | `codex exec --sandbox read-only --full-auto`        |
| Review existing changes                 | **review**    | `codex review`                                      |

- Default to **implement** if ambiguous and the task involves writing code.
- Default to **advise** if the user asks "how would you..." or wants options
  explored before committing.
- Default to **review** if the user asks for a code review or critique.

### Before running

1. Confirm `PROJECT_ROOT` is set.
2. **Implement mode only**: check `git status`. If the working tree is dirty,
   warn the user and confirm — Codex writes will mix with uncommitted changes.
3. Never use `--sandbox danger-full-access` unless the user explicitly
   requests full system access.

### Command templates

All Codex commands ensure `.scratch/` exists, redirect stdout to a dated
output file, and capture stderr for diagnostics. Always use `timeout: 600000`
on the Bash call. The output filename pattern is
`codex-{date}-{topic}.md` (e.g., `codex-2026-02-14-refactor-utils.md`).

If the task string contains double quotes, use a heredoc or escape them
(`\"`) so the shell parses the command correctly.

**Implement:**
```bash
mkdir -p "$PROJECT_ROOT/.scratch" && codex exec --sandbox workspace-write --full-auto -C "$PROJECT_ROOT" "<task>" 2>"$PROJECT_ROOT/.scratch/codex-stderr.log" > "$PROJECT_ROOT/.scratch/codex-{date}-{topic}.md"
```

**Advise:**
```bash
mkdir -p "$PROJECT_ROOT/.scratch" && codex exec --sandbox read-only --full-auto -C "$PROJECT_ROOT" "<task>" 2>"$PROJECT_ROOT/.scratch/codex-stderr.log" > "$PROJECT_ROOT/.scratch/codex-{date}-{topic}.md"
```

**Review:**
```bash
mkdir -p "$PROJECT_ROOT/.scratch" && codex review -C "$PROJECT_ROOT" 2>"$PROJECT_ROOT/.scratch/codex-stderr.log" > "$PROJECT_ROOT/.scratch/codex-{date}-{topic}.md"
```

Review scope defaults to all uncommitted changes. Scope it with flags and an
optional prompt argument:
```bash
# Uncommitted changes (explicit, same as default)
mkdir -p "$PROJECT_ROOT/.scratch" && codex review --uncommitted -C "$PROJECT_ROOT" 2>"$PROJECT_ROOT/.scratch/codex-stderr.log" > "$PROJECT_ROOT/.scratch/codex-{date}-{topic}.md"

# Changes relative to a branch
mkdir -p "$PROJECT_ROOT/.scratch" && codex review --base main -C "$PROJECT_ROOT" 2>"$PROJECT_ROOT/.scratch/codex-stderr.log" > "$PROJECT_ROOT/.scratch/codex-{date}-{topic}.md"

# A specific commit
mkdir -p "$PROJECT_ROOT/.scratch" && codex review --commit abc1234 -C "$PROJECT_ROOT" 2>"$PROJECT_ROOT/.scratch/codex-stderr.log" > "$PROJECT_ROOT/.scratch/codex-{date}-{topic}.md"

# Scoped with a prompt and a title
mkdir -p "$PROJECT_ROOT/.scratch" && codex review --base main --title "Auth refactor" -C "$PROJECT_ROOT" "Focus on error handling and edge cases" 2>"$PROJECT_ROOT/.scratch/codex-stderr.log" > "$PROJECT_ROOT/.scratch/codex-{date}-{topic}.md"
```

Stderr is captured to `codex-stderr.log` (not discarded). On non-zero exit,
check the stderr log for diagnostics. Stdout is redirected to the dated output
file to keep Codex output out of agent context.

This output file is internal to the executing agent. Only the bridge file (see
Bridge handoff below) is referenced in downstream delegation prompts.

### Long-running tasks

If the task is likely to exceed 10 minutes (large refactors, full-project
reviews), use `run_in_background: true` on the Bash call instead of blocking.
Check back with the user when it completes.

### Task description

Write a clear, self-contained task description. Include:
- What to do (the goal)
- Which files or modules are involved
- Any constraints (language, style, dependencies, approach)
- What NOT to change, if relevant

Do not dump raw conversation context. Distill the task into a focused
prompt — Codex has no prior context.

### Model selection

Use Codex's default model unless the user specifies otherwise.
Pass `--model <model>` only when explicitly requested.

### After running

**Context budgeting**: if the Codex output or git diff exceeds 200 lines,
read the first 50 and last 50 lines rather than the full file.

**Implement mode:**
1. Capture the diff:
   ```bash
   mkdir -p "$PROJECT_ROOT/.scratch" && git -C "$PROJECT_ROOT" diff > "$PROJECT_ROOT/.scratch/codex-last-diff.md"
   ```
   Scan selectively (same 200-line rule). Do not load the entire diff into
   context.
2. Summarize: files modified, nature of changes, anything unexpected.
3. If the diff is large, focus on structural changes, not line-by-line.

**Advise mode:**
1. Summarize Codex's suggestions from the output file.
2. Note where you agree or disagree — apply your own judgment.

**Review mode:**
1. Summarize findings from the output file.
2. Categorize as: issues found, suggestions, positive observations.

### Error handling

- Non-zero exit: check `$PROJECT_ROOT/.scratch/codex-stderr.log` for
  diagnostics, then report the failure. Do NOT retry automatically.
- Sandbox limit hit: report what Codex tried to do, ask the user whether
  to escalate (e.g., read-only -> workspace-write).
- Empty or garbled output: say so. Do not fabricate a summary.

## Output format

When invoked directly by the user, display results inline:

```
## Codex result

**Mode**: implement | advise | review
**Sandbox**: read-only | workspace-write
**Exit**: success | failure (reason)

### Summary
[2-4 sentences: what Codex did or said]

### Changes (implement mode only)
[Files modified and what changed in each]

### Assessment
[Your evaluation — agreements, disagreements, concerns]

### Next steps
[Test, review, iterate, or done]
```

## Bridge handoff (sub-agent context)

When invoked within a delegated sub-agent workflow (not directly by the
user):

1. Redirect Codex output to `$PROJECT_ROOT/.scratch/codex-{date}-{topic}.md`
   (already done by the command templates above).
2. Write Claude's assessment to
   `$PROJECT_ROOT/.scratch/bridge-{date}-codex-{topic}.md` with the first
   line: `<!-- Ephemeral bridge file — valid this session only -->`
3. Return a decision-grade summary to main: Did it succeed? Anything
   unexpected? Should the plan change?

Do not display the inline output format in this case.

## Critical evaluation

Treat Codex output as a colleague's PR, not as authoritative. If the
implementation looks wrong, say so. If suggestions conflict with project
conventions or existing patterns, flag it. If you would do something
differently, explain why. Codex is a second opinion — engage with it
critically.

## Session resume [experimental]

To continue a previous Codex session (e.g., iterate on its approach):
```bash
mkdir -p "$PROJECT_ROOT/.scratch" && echo "<follow-up prompt>" | codex exec resume --last 2>"$PROJECT_ROOT/.scratch/codex-stderr.log" > "$PROJECT_ROOT/.scratch/codex-{date}-{topic}.md"
```

This feature is experimental and may not work reliably. If resume fails,
re-run with a fresh task description that references the previous output:
```bash
mkdir -p "$PROJECT_ROOT/.scratch" && codex exec --sandbox workspace-write --full-auto -C "$PROJECT_ROOT" \
  "Continue from the approach in .scratch/codex-{date}-{topic}.md: <follow-up>" \
  2>"$PROJECT_ROOT/.scratch/codex-stderr.log" > "$PROJECT_ROOT/.scratch/codex-{date}-{topic}-followup.md"
```

## Permissions summary

| Sandbox mode         | What Codex can do                              | When to use                        |
|----------------------|-------------------------------------------------|------------------------------------|
| `read-only`          | Read files only, no writes                      | Advise, review, analysis           |
| `workspace-write`    | Read + write within the project directory only  | Implementation tasks (default)     |
| `danger-full-access` | Unrestricted system access                      | Never, unless user explicitly asks |

Escalation path: if Codex fails due to sandbox limits -> report -> ask
user -> re-run with elevated sandbox if approved.

Codex picks up conventions from `~/.codex/AGENTS.md` and any project-local
`AGENTS.md` or `CLAUDE.md` automatically. Do NOT inject conventions into
the task prompt.
