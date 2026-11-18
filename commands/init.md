---
description: Create this project's two devil config files (critical-path globs and invariants) from the plugin's templates — never overwriting existing ones — and check the prerequisites.
---

Set up the devil's per-project configuration in this repository. Work from the
repo root.

1. **Copy the templates, never overwriting.** For each pair, copy only when
   the target does not exist yet, and say which ones were already there:

   - `${CLAUDE_PLUGIN_ROOT}/templates/critical-globs.conf.example` →
     `.claude/devil/critical-globs.conf`
   - `${CLAUDE_PLUGIN_ROOT}/templates/critical-invariants.md.example` →
     `.claude/devil/critical-invariants.md`

   (`mkdir -p .claude/devil`, then `cp -n <template> <target>` for each.)

2. **Fill them in with the user — do not invent rules.**
   - Globs: propose the modules that look critical in this codebase (money,
     auth, data deletion, migrations, anything irreversible), one glob per
     line, and let the user confirm or correct the list.
   - Invariants: ask the user for the project's catastrophic, non-negotiable
     rules, or lift them from the rules already written in CLAUDE.md and
     AGENTS.md. Write them as instructions to a hostile reviewer and remove
     the `EDIT-ME` line — nothing is injected while it is there. If the user
     has none yet, leave the template as it is and say that critical-path
     injection stays off until it is filled in.
   - Remind the user that codex reads AGENTS.md at the repo root as repo law:
     the same invariants belong there and in CLAUDE.md.

3. **Check the prerequisites and report each:** `git`, `jq` and `codex` on
   PATH (without codex every devil run is a clean no-op; `codex login` signs
   in), and `setsid` or `perl` plus `nohup` for `--detach`.

4. **Verify.** Run `DEVIL_DRY_RUN=1 devil.sh diff` (it spends nothing) and
   report any `devil.sh: note:` line it prints: a note means critical-path
   detection or invariant injection is still off, and says why. Then run
   `devil.sh budget` and report the `DEVIL_QUOTA:` line.

Both files are data: they are read line by line and matched, never executed.
Commit them with the project so every teammate's devil judges by the same
rules.
