# WORKFLOW — the operating model

The plugin's pieces implement one discipline. This file is the map; the
enforcement lives in your project's CLAUDE.md and AGENTS.md (the rules) and in
the plugin's commands, agents and `devil.sh` (the machinery).

## Roles

| Role | Who | Writes code? | Commits? |
|---|---|---|---|
| **Supervisor** | The main Claude session (strongest model) | Rarely — tiny/critical edits only | **Always and only the supervisor** |
| **Expert** | `/devil:expert` subagent (strong model, max effort, full tools) | Yes — from a self-contained brief | Never |
| **Angel** | `/devil:angel` subagent (strongest model, read-only by instruction) | Never | Never |
| **Devil** | codex via `devil.sh` (sandbox-enforced read-only) | Never | Never |
| **Domain-auditor** | subagent (read-only by instruction) | Never | Never |

Angel **advises** (recommendation-led, blunt); devil **attacks** (objections
only, verdict line). They are deliberately different vendors/models so their
blind spots don't correlate. On high-stakes work run both — but **not in
equal measure**: the angel costs only the session's own quota, the devil
spends a scarce external one (see "Spending the devil" below). The angel
carries the multi-round argument; the devil is one terminal shot.

## The loop for a nontrivial task

1. **Plan.** Write a self-contained brief to the scratchpad — not into the
   repo. It opens with its **premise**: ONE claim about the current system
   the plan rests on, plus the existing mechanisms already searched BY
   CAPABILITY (what does this job today, whatever it is called), each with
   the grep used — then the goal, file-by-file intent, invariants touched,
   test plan, open questions. Both critics re-run those greps first; a plan
   that duplicates something already there is the cheapest objection to
   find, and `devil.sh` refuses a plan brief with no premise heading.
2. **Critique to closure.** Run the Claude lane first and in full
   (`/devil:angel plan`, your own verification), arguing as many rounds as the
   decision needs; then `/devil:devil plan <brief>` ONCE (launched `--detach`; the
   marker the latest revision folded in goes to `--expect`, so a stale or
   unsaved brief is never attacked), as the terminal
   cross-vendor check on the settled plan. Answer EVERY numbered objection
   explicitly: `CONCEDE — #n: <fix>` or `REBUT — #n: <file:line evidence>`.
   **Closure is disposition completeness, not a verdict** — the devil's
   preamble forbids praise, so PROCEED is near-unreachable and a loop
   waiting for it never terminates. **While any critic still BLOCKs, no
   stage of the plan is settled — dispatch nothing.**
3. **Weigh, don't obey.** A finding is an argument, not an instruction —
   sort into fix / accept-risk-explicitly / drop by real failure cost
   before adding machinery.
4. **Dispatch.** `/devil:expert` with a self-contained brief (the expert sees no
   conversation context). Long tasks in background; corrections to a
   running/finished expert go via SendMessage to the SAME agent id — never
   a respawn.
5. **Review.** Supervisor reads the FULL diff file-by-file, runs the
   project gates; on critical-path diffs also the domain-auditor and the
   harness's own code-review/security-review passes. `/devil:devil diff` is
   discretionary — spend it when the design was NOT devil-closed, or when
   the implementation departed from the closed plan; otherwise say in the
   commit body that it was skipped and why.
6. **Commit.** Supervisor only — when asked, or at the work-unit
   boundaries an explicitly autonomous run's ground rules authorize.
   Conventional Commits, subject ≤ 50, body wrapped at 72 via
   `git commit -F <file>`.
7. **Wrap up.** Before declaring a multi-step agenda done: `/devil:angel wrapup`
   against a checklist summary with evidence pointers — and `/devil:devil wrapup`
   at the end of a *critical* arc, not at the end of every agenda.

## Defense on critical paths

The plugin ships the review-time layer. The other two are yours to add, and
worth adding:

1. **Authoring time** *(not shipped)* — a PostToolUse hook that flags banned
   patterns on added lines in critical files the moment they are written (a
   heuristic reminder, not an authority).
2. **Review time** *(shipped)* — the domain-auditor plus the devil, with
   auto-escalated effort and the invariants file injected, on every
   critical-path diff.
3. **Regression time** *(not shipped)* — invariant tests run on every commit
   and in CI. Every stated "must not" gets its regression test in the same
   change. A "must not" that a script can check, run on every commit, never
   needs a critic again.

One data file (`.claude/devil/critical-globs.conf`) scopes the critical
paths; point your own hooks and checks at the same file so the layers cannot
drift apart. Coverage of the globs themselves is layer 3's job: an
anchor-based completeness test proving every critical file is matched.
Matching lists only guarantee *agreement* — a critical file outside the list
evades every layer until an anchor test pins coverage.

## The critic contract (both directions)

- The critic verifies against the actual repo, numbers objections
  strongest-first — one independent defect per number, each a one-line claim
  plus an `Evidence:` line (file:line or a diff hunk) and a `Why:` line, with
  no headings, tables or emphasis — and ends with an undecorated
  `VERDICT: BLOCK | FIX-FIRST | PROCEED` line.
- Every run closes with four machine lines — `DEVIL_RUN:` (which mode, base,
  criticality, effort and the model the script chose for it), `DEVIL_QUOTA:`
  (what it left in the tank), `DEVIL_SESSION:` (the uuid a reply would need),
  and `DEVIL_LOG:`. Quote the first three when reporting a critique: a verdict
  whose configuration is unknown cannot be weighed later. A run that fails
  prints ONE `DEVIL_FAILED: <reason>` line as its last line instead (a
  refusal, a codex error exit with codex's own reason, a TERM). Before
  those: `DEVIL_PREFLIGHT:` lines (model offered? weekly window ≥ the run's
  cost + 1?), `DEVIL_START:` (codex was invoked) and, for plan and wrapup,
  `DEVIL_BRIEF:` (sha256, size and mtime of the bytes attacked). Launched
  with `--detach`, the log's LAST line decides: `DEVIL_LOG:` = done,
  `DEVIL_FAILED:` = failed; `kill -TERM <DEVIL_PID>` cancels. The model
  follows what the run is worth (plan and critical-path runs: the strong
  lane; the rest: the cheap lane).
- The critic also states what it could NOT verify (`UNVERIFIED:`), and
  treats anything you report as already-fixed as a claim to check in the
  code rather than settled ground.
- You answer every objection — concede-with-fix or rebut-with-evidence —
  and never silently drop one. A second round is a FRESH run against the
  updated artifact carrying the prior objections + dispositions, not a
  `reply`: a reply replays the whole session and costs about twice as much.
  Keep `reply <session-id>` (the exact `DEVIL_SESSION:` uuid; `--last` is
  deliberately unsupported) for when the answer itself needs attacking.
- The devil is critic-ONLY. The moment it writes code, its critiques stop
  being independent ("it would literally become the devil itself").
- Rule changes must reach the critic: CLAUDE.md ↔ AGENTS.md ↔
  `critical-invariants.md` sync in the same commit.

## Spending the devil

The devil runs on an exhaustible external quota, so *how* it is spent
matters as much as *that* it runs. Each codex run records the account's
rate-limit window in its own `~/.codex/sessions/**/rollout-*.jsonl`, so the
spend is measurable after the fact — `devil.sh budget` reads the current
window for free. The source project measured it twice, on two plans:

**The Pro re-baseline (2026-09-19, n=7 runs).** After the account moved to
ChatGPT **Pro**, the per-run share of the weekly cap fell by roughly a factor
of three — a `plan` run on the strong model lane costs about **one point of
the weekly cap** (6–7 minutes), a critical-path `diff` about two, and the one
cheap-lane mode measured was ~0:

| Mode | Model lane | Avg cost, points of a weekly cap | Wall clock |
|---|---|---|---|
| `plan` | strong | ~1 | 6–7 min |
| `diff`, critical-path | strong | ~2 | — |

Seven runs is a small sample and is stated as one; it is enough to settle the
one question the spend policy turns on — whether a plan round costs a percent
of the week or several. It costs about a percent. This is also the pricing
`devil.sh`'s pre-flight uses: it refuses a run unless the weekly window shows
the run's cost **plus one point** (plan 2%, diff/wrapup 2%, critical 3%, reply
5%), at any reading age. Take your own reading before rationing anything —
`devil.sh budget` before and after a run — and remember that cheaper per run
is not free.

**The Plus measurement (2026-08, 184 runs overall).** Kept because the three
findings below are drawn from it, and they still hold — the Pro re-baseline
moved the price, not the shape:

| Mode | Avg cost, points of a weekly cap (Plus) | Share of a month's spend |
|---|---|---|
| `plan` | ~2.9 | 8% |
| `diff` | ~4.3 | 32% |
| `reply` | ~8.2 | **59%** |

Three findings, and what each one implies:

1. **The reply loop is the drain, not the devil** — 59% of a month across 26
   rounds. A resume re-sends the accumulated session *and* re-explores the
   repo. One 7-round arc spent 64 points of a 100-point week. → One shot per
   decision point; a second round is a fresh run, not a reply.
2. **The verdict carries almost no signal** — 80 runs returned FIX-FIRST 59,
   BLOCK 18, PROCEED 3. That is by construction: the preamble forbids
   praise. → Close on dispositions, never on a verdict. Note that `plan` is
   where BLOCK concentrates (7 of 13 plan runs vs 5 of 29 diff runs), which
   is why plan is the mandated pass: design rejection is what a same-vendor
   critic is least likely to raise and the most expensive to find late.
3. **You pay for the reading, not the thinking** — a diff run consumes
   5–14M input tokens and emits ~20k; raising reasoning effort adds ~10–25k,
   under 0.5% of the run. → Escalate every run by default (a model's own
   default reasoning level may be `low`, so don't rely on the config
   baseline), and narrow what it reads with `-- <paths>`.

The corollary is a two-lane rule worth writing into your CLAUDE.md:
everything the Claude lane
can find, it must find *before* codex is spent, so the devil's one run buys
residual cross-vendor risk rather than re-discovering what a local critic
already had. (On the Pro numbers the single terminal shot is no longer forced
by budget; the Claude-lane-first order stays, because it is what makes the
devil's run worth its cost.)

The model follows the same logic: `devil.sh` gives `plan` and critical-path
runs the **strong** lane — design rejection and critical-path reasoning are
the findings most expensive to find late — and every other run the **cheap**
lane. The default slugs are the source project's; point the lanes at what
your account offers with `DEVIL_MODEL_CRITICAL` / `DEVIL_MODEL_DEFAULT` (the
pre-flight refuses a model codex's own cache does not list, and says which it
does).

## Growing the setup

Start with what ships here. Add machinery only when a real failure justifies
it (materiality rule) — the source project's additions each trace to a
specific incident: the tripwire to a slipped pattern, the completeness test
to a file that evaded review, the quota bridge to a mid-response cutoff.
When you add a rule, add its regression test and its AGENTS.md mirror in the
same commit — and consider saving the lesson as a memory so the next session
starts where this one ended.
