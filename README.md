# devil

Adversarial review for Claude Code: before a plan is built or a change is
called done, someone attacks it.

- **The devil** is the OpenAI codex CLI, run as a read-only critic from a
  different vendor, so its blind spots do not match Claude's. It attacks a
  plan, the working diff or a completion claim, and Claude must answer every
  numbered objection: concede with a fix, or rebut with evidence.
- **The angel** is a strongest-model Claude subagent that gives blunt counsel
  first and carries the multi-round argument, because it costs only your
  session quota.
- **The expert** is a strong-model implementer you delegate to. It never
  commits; the session reviews its diff and makes the commit.

## What you get

| Command | What it does |
|---|---|
| `/devil:devil [budget\|plan\|diff\|wrapup\|reply <id>\|focus]` | Runs codex against a plan brief, the diff or a "done" claim, then answers every objection |
| `/devil:angel [plan\|diff\|stuck\|wrapup\|question]` | Asks the angel for a recommendation, objections and flags before you act |
| `/devil:expert <task>` | Delegates a task to the expert from a self-contained brief |
| `/devil:init` | Creates the project's two config files from templates |

| Agent | Role |
|---|---|
| `devil:angel` | Advisor, read-only by instruction |
| `devil:expert` | Implementer, full tools, never commits |
| `devil:domain-auditor` | Audits critical-path diffs against your declared invariants, read-only by instruction |
| `devil:test-writer` | Writes tests to your conventions; owns the "must not ⇒ regression test" duty |

`devil.sh` is the one script behind `/devil:devil`. The plugin puts it on the
Bash tool's PATH, so Claude runs it as a bare command.

## Install

At the prompt of a terminal session:

```
/plugin install devil --marketplace boiled-darvari/claude-code-devil
```

From a shell:

```
claude plugin install devil --marketplace boiled-darvari/claude-code-devil
```

In the desktop app, where `/plugin` does not run: click + next to the prompt
box, then Plugins, Add marketplace with `boiled-darvari/claude-code-devil`,
then Add plugin.

## Requirements

- `git`, `bash` and `jq`.
- The **codex CLI**, signed in with `codex login`. Without it every devil run
  is a clean no-op that says so; the angel, the expert and the auditor work
  regardless.
- `nohup` plus `setsid` (or `perl`, as on macOS) for the detached launch that
  `/devil:devil` uses.

## Set up a project

Run `/devil:init` in the repository. It creates two files and helps you fill
them in:

| File | What it holds |
|---|---|
| `.claude/devil/critical-globs.conf` | One glob per line: the paths you consider critical |
| `.claude/devil/critical-invariants.md` | The project's catastrophic, non-negotiable rules, written for a hostile reviewer |

A diff that touches a critical path takes the strong model lane, escalates
codex's reasoning effort and gets the invariants injected into the prompt.
Both files are data: read and matched, never executed. Until they are filled
in, the devil still runs and says on each run what is switched off.

Codex reads `AGENTS.md` and `CLAUDE.md` at the repo root as repo law, so keep
the same invariants there.

To stop the permission prompt for the script, add these to
`permissions.allow` in the project's `.claude/settings.local.json`:

```json
"Bash(devil.sh *)",
"Bash(DEVIL_DRY_RUN=1 devil.sh *)"
```

## How a run goes

1. Claude runs its own lane first: its own verification and `/devil:angel`.
2. `/devil:devil plan <brief>` (or `diff`, `wrapup`) launches codex detached,
   in a read-only sandbox. A pre-flight refuses the run before anything is
   spent if codex does not offer the chosen model or the weekly quota is too
   low.
3. Codex investigates the repository itself and returns numbered objections
   with `file:line` evidence and a `VERDICT:` line.
4. Claude answers every objection with `CONCEDE` or `REBUT`. Closure is every
   objection answered, not a `PROCEED` verdict.

`devil.sh budget` reads codex's own logs and prints the remaining quota for
free. [WORKFLOW.md](WORKFLOW.md) has the full operating model and the measured
cost of each mode.

## Models and knobs

`devil.sh` picks the model by what the run is worth: `plan` and any
critical-path run take the strong lane, every other run the cheap lane.

| Variable | Effect | Default |
|---|---|---|
| `DEVIL_MODEL` | Use this model for the run; beats both lanes | unset |
| `DEVIL_MODEL_CRITICAL` | The strong lane | `gpt-6-astra` |
| `DEVIL_MODEL_DEFAULT` | The cheap lane | `gpt-5.6-terra` |
| `DEVIL_EFFORT_MAX` | Reasoning effort every run escalates to | `xhigh` |
| `DEVIL_ESCALATE=0` | Drops a non-critical run back to your codex config's effort | escalation on |
| `DEVIL_DRY_RUN=1` | Prints the prompt and the codex argv; spends nothing | off |
| `DEVIL_FORCE=1` | Skips the pre-flight's model and quota checks | off |

If your codex account does not offer the default models, the pre-flight
refuses the run and lists the ones it does: point the lanes at those.

## What is enforced and what is only instructed

- **The devil is sandbox-enforced read-only.** `--sandbox read-only` is
  hardcoded in `devil.sh`; no flag, mode or environment variable widens it,
  and the tests pin that.
- **The angel and the domain-auditor are read-only by instruction.** Their
  tools are limited to Read, Grep, Glob and Bash, but Bash is not a sandbox.
- **The expert's "never commits" is by instruction too**, and it has full
  tools. The session reviewing every diff and making every commit is the
  actual control.

## Tests

```
bash tests/devil-sh-invariants.sh
```

The suite runs `devil.sh` against a stand-in for codex in a scratch
repository, so it spends no quota and touches no real codex state. It pins the
read-only sandbox, the argument and option-injection refusals, the failure
marker, the detached launch, the pre-flight and the model lanes.

## Not included

The plugin covers review and delegation. A commit gate, an authoring-time
tripwire hook and always-run invariant tests are worth adding to a project,
and WORKFLOW.md describes where they fit, but they are yours to write: their
settings are per project, and a plugin is one shared copy.
