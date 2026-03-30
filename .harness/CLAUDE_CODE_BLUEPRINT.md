# Claude Code Implementation Blueprint
## Native Translation of the Harness Engineering Playbook

**Blueprint version: 1.6.0** | **Playbook version: 3.4.0** | **Date: 2026-03-16**

> This document translates the tool-agnostic Harness Engineering Playbook into
> Claude Code's native control surfaces. The playbook remains the source of truth
> for concepts and evidence. This blueprint is the implementation guide for Claude Code.

**Cross-document policy override:** This blueprint intentionally overrides the playbook's
default cross-tool policy of AGENTS.md-as-canonical. For Claude Code, `CLAUDE.md` is
primary and `AGENTS.md` is the Codex compatibility mirror. This blueprint assumes the
bootstrap prompt and deployment guide have been updated to support `TARGET: claude-code`
branching. If they have not, update them before relying on this blueprint — the four
documents are only consistent when all use the TARGET parameter.

> **Operational theatre warning:** This blueprint describes a comprehensive harness
> with 10 subagents, observability contracts, canary protocols, and multi-agent
> architecture. **Do not deploy all of this at once.** A harness larger than what
> your team actually operates becomes stale faster than your code. Start with
> Pass 1 (CLAUDE.md + settings + initializer + verifier). Add components only
> when a real failure mode demands them. The harness should earn its complexity
> through demonstrated need, not pre-emptive governance.
>
> **The simplification principle applies from day one:** if a component is not
> actively used and maintained, it is not a safety control — it is a false
> sense of safety. Remove it or operate it. There is no middle ground.

---

## Design Principle

For Claude Code, **CLAUDE.md is canonical**. Generate AGENTS.md as the Codex compatibility mirror, not the reverse.

Claude Code's native surfaces are:

| Surface | Purpose | Playbook equivalent |
|---|---|---|
| `CLAUDE.md` | Persistent project instructions, auto-loaded | Root `AGENTS.md` pointer map |
| `.claude/settings.json` | Permissions, hooks, environment | Merge gates, safety rules, CI enforcement |
| `.claude/agents/` | Project subagents with restricted tools | Automations, specialized workflows |
| `.claude/skills/` | Optional local convention for skill-style packaging. Anthropic clearly documents subagents (`.claude/agents/`) and slash commands (`.claude/commands/`) as native surfaces; `.claude/skills/` has less official documentation. Use subagents or slash commands as the documented safer default. | Skills, large task-specific instructions |
| `.claude/commands/` | Slash commands for common workflows | Make target shortcuts |
| `docs/` | Architecture, golden principles, security | Same — unchanged |
| `scripts/` | Enforcement scripts, linters | Same — unchanged |
| `Makefile` | Command contracts | Same — unchanged |

---

## Directive Classification

Every directive from the playbook, classified by Claude Code surface.

### Bucket A — CLAUDE.md (always-loaded context)

These are workflow norms, pointers, and conventions the agent should always know.

```
FROM PLAYBOOK                              → CLAUDE.md SECTION
─────────────────────────────────────────────────────────────
Command map (all make targets)             → ## Commands
Architecture pointer                       → ## Architecture → docs/ARCHITECTURE.md
Golden principles pointer                  → ## Conventions → docs/GOLDEN_PRINCIPLES.md
Merge gates pointer                        → ## Merge Gates → docs/MERGE_GATES.md
Security pointer                           → ## Security → docs/SECURITY.md
Testing pointer                            → ## Testing → docs/TESTING.md
Code Navigation Rule (orient→locate→read)  → ## Code Navigation Rule
Loop Breaker (3 cycles → stop → escalate)  → ## Loop Breaker
Anti-laziness (verify before claiming done) → ## Anti-Laziness
Git coordination (commit after every unit)  → ## Git Coordination
Test oracle (continuous reference testing)  → ## Test Oracle
Initializer instructions (first/subsequent)→ ## Session Start
Tool preference order (make > shell)       → ## Tool Hierarchy
Tool output offloading rule (>200 lines)   → ## Output Management
Definition of Done (summary + pointer)     → ## Done Criteria → docs/MERGE_GATES.md
Current knowledge access rule              → ## External Knowledge
Metrics queryability by CLI                → ## Commands (make metrics-*)
Walkthrough video for human review         → ## Commands (make record-walkthrough)
Continuous improvement / lessons pipeline  → ## Continuous Improvement
Agent memory capture triggers (§1.8)       → ## Agent Memory
Context management (resets vs compaction)  → ## Context Management
Multi-agent file communication protocol   → ## Multi-Agent Communication
Sprint contracts (per-task negotiated DoD) → ## Sprint Contracts
```

### Bucket B — .claude/settings.json rules (path-specific policy)

Claude Code's officially documented native surfaces are `CLAUDE.md` (with imports
and hierarchical loading), `.claude/settings.json`, `.claude/agents/`, and `.claude/commands/`.
(`.claude/skills/` is an optional local convention with less official documentation —
see the design table at top.) For modular per-domain rules, use **directory-level CLAUDE.md
files** (e.g., `src/auth/CLAUDE.md`) or CLAUDE.md imports via `@path` syntax.

```
FROM PLAYBOOK                              → IMPLEMENTATION
─────────────────────────────────────────────────────────────
Per-subsystem AGENTS.md overrides          → src/auth/CLAUDE.md, src/billing/CLAUDE.md, etc.
Auth-sensitive rules                       → src/auth/CLAUDE.md + permission deny rules
Infra rules                                → infra/CLAUDE.md + permission deny rules
Frontend proof requirements                → src/ui/CLAUDE.md
Docs freshness rules                       → docs/CLAUDE.md
Protected paths (globs)                    → settings.json permission deny rules
```

### Bucket C — .claude/settings.json hooks (deterministic runtime enforcement)

These are the rules that must be mechanically enforced, not just instructed.

```
FROM PLAYBOOK                              → HOOK EVENT
─────────────────────────────────────────────────────────────
Session state display + init check         → SessionStart
Protected-path write review               → Permission ask rules (Edit(src/auth/**) etc.) + PostToolUse flagging
Deny direct merge/deploy commands          → PreToolUse (matcher: "Bash") + deny rules
Deny broad file dumps (cat **/*.ts)        → PreToolUse (matcher: "Bash")
Deny reads of .env / secrets               → Permission deny rules (no hook needed)
Post-edit scoped lint/type check           → PostToolUse (matcher: "Edit|Write")
Post-doc-edit freshness validation          → PostToolUse (matcher: "Edit|Write")
Protected-path edit flagging               → PostToolUse (matcher: "Edit|Write")
Definition of Done gate on stop       → Stop (fast state-bound checks: progress.json
                                              + verifier stamp bound to git HEAD +
                                              dirty-files hash; full DoD verification
                                              delegated to verifier subagent)
progress.json update enforcement            → Stop
```

### Bucket D — .claude/agents/ (specialized subagents)

These are the playbook's automations and specialized workflows, each
with a single clear responsibility and restricted tool access.

**Deploy incrementally, not all at once.** Start with CORE subagents only.
Add OPTIONAL subagents when a real failure mode demands them.

```
CORE SUBAGENTS (deploy in Pass 1–3, always needed):
  Initializer Run (§1.5)                     → .claude/agents/initializer.md
  Definition of Done / proof verifier         → .claude/agents/verifier.md

STANDARD SUBAGENTS (deploy in Pass 4, recommended for active projects):
  docs-gardener automation                    → .claude/agents/docs-maintainer.md
  Merge gate checker / Land skill             → .claude/agents/merge-checker.md
  Memory consolidation / dream cycle (§1.8)   → .claude/agents/dream-sweep.md

OPTIONAL SUBAGENTS (deploy only when a specific need arises):
  golden-principle-sweep                      → .claude/agents/golden-sweep.md
  Security review for protected paths         → .claude/agents/security-reviewer.md
  quality-score-update                        → .claude/agents/quality-scorer.md
  Continuous improvement / lessons promotion  → .claude/agents/lessons-sweep.md
  Planner: prompt → full spec (§2.7)          → .claude/agents/planner.md
  Generator-Evaluator loop (§2.7)             → .claude/agents/evaluator.md
```

> **Simplification applies here too:** If a subagent is deployed but never invoked
> over 2+ weeks of active development, it is dead weight. Remove it and re-add
> only when a real incident justifies it.

---

## File Manifest: Claude Code Edition

```
your-repo/
├── CLAUDE.md                              ← PRIMARY root map (< 100–200 lines)
├── AGENTS.md                              ← Codex compatibility mirror (generated from CLAUDE.md)
├── WORKFLOW.md                            ← orchestration contract (optional, §3.4 of playbook)
├── .claude/
│   ├── settings.json                      ← hooks, permissions, environment
│   ├── settings.local.json                ← personal overrides (gitignored)
│   ├── traces/                            ← per-run JSONL trace files (append-only)
│   │   └── run-<uuid>.jsonl               ← immutable event log per session
│   ├── tasks/                             ← task-scoped runtime state (by branch name)
│   │   └── <task-id>/
│   │       ├── current-run-id             ← run_id for this task's current session
│   │       ├── progress.json              ← structured task state (scoped, not global)
│   │       └── evidence-bundle.json       ← latest verifier evidence bundle
│   ├── memory/                            ← agent memory (Auto Dream compatible)
│   │   ├── MEMORY.md                      ← index file (< 200 lines, one-line pointers)
│   │   ├── dream-state.json               ← consolidation trigger state
│   │   └── <topic>.md                     ← topic files (auto-created during sessions)
│   ├── agents/
│   │   ├── initializer.md                 ← first-session bootstrap agent
│   │   ├── verifier.md                    ← proof & DoD verification agent
│   │   ├── docs-maintainer.md             ← documentation gardening agent
│   │   ├── golden-sweep.md                ← golden principle enforcement agent
│   │   ├── merge-checker.md               ← merge gate verification agent
│   │   ├── security-reviewer.md           ← protected-path review agent
│   │   ├── quality-scorer.md              ← domain quality assessment agent
│   │   ├── lessons-sweep.md              ← lesson promotion agent
│   │   ├── dream-sweep.md               ← memory consolidation agent (Auto Dream pattern)
│   │   ├── planner.md                    ← prompt → full spec expansion agent
│   │   └── evaluator.md                  ← adversarial QA / grading agent
│   ├── skills/                            ← OPTIONAL local convention (less official docs)
│   │   └── harness-audit/
│   │       └── SKILL.md                   ← on-demand harness compliance check
│   └── commands/
│       ├── audit-harness.md               ← /audit-harness slash command
│       ├── init-task.md                   ← /init-task slash command
│       └── promote-lessons.md            ← /promote-lessons slash command
├── .claude/handoffs/                      ← inter-agent file communication
│   ├── spec.md                            ← planner → generator (product spec)
│   ├── sprint-<n>-contract.md             ← generator ↔ evaluator (negotiated)
│   ├── sprint-<n>-feedback.md             ← evaluator → generator (grading)
│   └── session-<n>-handoff.md             ← context reset handoff artifacts
├── src/
│   ├── auth/CLAUDE.md                     ← auth-domain rules (< 50 lines)
│   ├── billing/CLAUDE.md                  ← billing-domain rules
│   └── ui/CLAUDE.md                       ← frontend-specific rules + proof reqs
├── infra/CLAUDE.md                        ← infra-domain rules
├── docs/
│   ├── ARCHITECTURE.md
│   ├── GOLDEN_PRINCIPLES.md
│   ├── MERGE_GATES.md
│   ├── SECURITY.md
│   ├── TESTING.md
│   ├── ROLLBACK.md
│   ├── INITIALIZER.md
│   ├── QUALITY_SCORE.md
│   ├── RELIABILITY.md
│   ├── OBSERVABILITY.md                   ← failure codes, correlation schema, evidence format
│   ├── CANARY_TASKS.md                    ← canary task corpus for harness regression testing
│   ├── design-docs/                       ← architectural decision records
│   ├── product-specs/                     ← feature specifications
│   ├── exec-plans/
│   │   ├── active/                        ← in-flight plans
│   │   ├── completed/                     ← done, kept for agent context
│   │   └── tech-debt-tracker.md
│   ├── generated/
│   │   └── lessons.md                     ← recurring mistakes and recovered heuristics
│   ├── references/                        ← external docs reformatted for agents
│   └── skills/land/SKILL.md
├── scripts/
│   ├── complexity-check.sh
│   ├── lint-no-duplicate-helpers.sh
│   ├── lint-boundary-validation.sh
│   ├── lint-schema-drift.sh
│   ├── lib/
│   │   ├── portable-hash.sh               ← single source of truth for hashing
│   │   └── portable-uuid.sh               ← single source of truth for run_id generation
│   ├── hooks/
│   │   ├── session-start.sh
│   │   ├── pre-tool-use-guard.sh
│   │   ├── post-edit-validate.sh
│   │   └── stop-gate.sh
│   └── ...
├── .harness/
│   ├── PLAYBOOK_SKILL.md                  ← full playbook (reference, not runtime)
│   └── harness.meta.json                  ← version stamp + audit log
├── progress.json                          ← backward compat only; prefer .claude/tasks/<id>/progress.json
├── init.sh
├── Makefile
└── proof/
```

---

## Implementation Details

### 1. CLAUDE.md Template

```markdown
# CLAUDE.md — Project Harness Map

## Commands
- `make dev`              → boot local dev environment
- `make test`             → run full test suite
- `make lint`             → run all linters including architectural checks
- `make project-summary`  → orient before source navigation
- `make find-refs SYMBOL=x` → AST/LSP symbol lookup
- `make check-docs`       → validate doc freshness and cross-links
- `make capture-dom-snapshot` → capture DOM state for proof (UI repos)
- `make record-walkthrough`   → capture UI video for human review (UI repos, optional)
- `make test-stress`      → determinism check on touched tests
- `make generate-service NAME=x DOMAIN=y` → scaffold in correct layer
- `make metrics-throughput`  → time-to-merge, PRs/day, iteration count
- `make metrics-quality`     → CI pass rate, defect escape rate, rollback frequency
- `make metrics-harness`     → doc freshness score, boundary violations, flake rate
- `make metrics-safety`      → blocked egress, secret-scan hits, permission denials

## Architecture
→ docs/ARCHITECTURE.md

## Conventions & Mandatory Patterns
→ docs/GOLDEN_PRINCIPLES.md

## Merge Gates & Done Criteria
→ docs/MERGE_GATES.md (single authority for all merge policy)

## Security
→ docs/SECURITY.md

## Testing
→ docs/TESTING.md

## Code Navigation Rule
1. Orient: `make project-summary`
2. Locate: `make find-refs SYMBOL=x` / `grep -rn` / ctags
3. Read: targeted file:line ranges only
Never dump broad file contents to "understand the repo."

## Tool Hierarchy
1. Use `make <target>` when a command contract exists.
2. Use `make find-refs`, `make project-summary` for navigation.
3. Fall back to shell commands in the worktree for uncovered tasks.

## Output Management
When a tool output exceeds ~200 lines or is not immediately needed in full:
1. Write the full output to a file (proof/, logs/, or docs/generated/).
2. Keep only a compact summary or head/tail excerpt in-context.

## Loop Breaker
If any fix-and-rerun cycle fails 3 times, STOP.
Document the dead-end, label the PR "blocked", escalate to a human.

## Anti-Laziness
Do NOT stop prematurely. When you think you're done:
1. Run the FULL test suite, not just the file you changed.
2. Check ALL acceptance criteria, not just the first one.
3. If a success criterion exists, verify you actually meet it.
4. Ask: "Am I really done, or finding an excuse to stop?"

## Git Coordination
Commit and push after every meaningful unit of work.
Run make test before every commit. Never commit code that breaks passing tests.
Clear commit messages: what changed, why, how validated.

## Test Oracle
When a reference implementation or test suite exists:
- Run tests continuously as you work, not just at the end
- Track accuracy in the progress file over time
- Record failed approaches in memory (with WHY) to prevent re-attempts

## Agent Memory
Memory lives in .claude/memory/ (separate from this instruction file).
MEMORY.md is the index (< 200 lines). Topic files hold the detail.

Write to memory when:
- A user corrects you (write the correction, not the mistake)
- You discover a recurring pattern (3+ occurrences)
- An important decision is made (architecture, library, API design)
- A debugging insight is found (root cause of a non-obvious bug)
- A build/deploy quirk is discovered

Do NOT write: transient task state (use progress.json), full code
snippets (point to file:line), speculation, or anything already here.

## Continuous Improvement
When a mistake is corrected (by you or by human feedback):
1. Write the correction to .claude/memory/ (agent memory)
2. Also record the lesson in docs/generated/lessons.md
3. After 5 lessons accumulate, promote to: golden principle, hook, verifier rule, or command contract
4. Mark promoted lessons with a reference to the artifact they became

## Session Start
If this is the FIRST session on a new task:
  Use the initializer subagent to set up the environment.
  It creates init.sh, progress.json, and the initial git commit.

If this is a SUBSEQUENT session:
  1. Read .claude/memory/MEMORY.md to orient on accumulated knowledge
  2. Read progress.json to understand current state
  3. Review git log since last session
  4. Pick the next "pending" feature — or resume the one "in_progress"
  5. Work on ONE feature at a time. Never have two "in_progress".
  6. Mark "completed" ONLY after tests + verification pass.
  7. Update progress.json before session ends.

If this looks like a CRASH RECOVERY (in_progress features + no recent handoff):
  1. Read progress.json — identify what was in_progress
  2. Run `git log --oneline -10` — find last committed state
  3. Run `make test` — check if the codebase is in a passing state
  4. If tests pass: the crash happened after a good state. Resume from progress.json.
  5. If tests fail: the crash happened mid-work. Check git stash and recent diffs.
     Roll back to last passing commit if necessary: `git reset --hard <last-passing>`
  6. Update progress.json with crash recovery notes in known_issues[].
  7. Record the crash in memory: what was lost, what was recovered, what to avoid.

## Before Ending a Session
Before stopping, ALWAYS use the verifier subagent.
The Stop hook will block session end if the verifier was not invoked.

## Context Management
If context is getting large and you notice declining coherence:
- Prefer compaction if the model handles it without premature wrap-up.
- If context anxiety appears (rushing to finish, skipping steps):
  write a handoff artifact to .claude/handoffs/session-<n>-handoff.md
  covering: what was done, current state, what to do next, open questions.
  The next session reads this + progress.json + recent git log.

## Multi-Agent Communication
Agents communicate via files in .claude/handoffs/:
- spec.md: planner → generator (product spec)
- sprint-<n>-contract.md: generator ↔ evaluator (negotiated DoD per sprint)
- sprint-<n>-feedback.md: evaluator → generator (grading + critique)
- session-<n>-handoff.md: outgoing → incoming agent (context reset)
Never delete handoff files — they are the narrative audit trail.

## Sprint Contracts
For complex features, negotiate a sprint contract with the evaluator before building:
1. Propose what you will build + testable acceptance criteria
2. The evaluator reviews and may request changes
3. Build against the agreed contract
4. The evaluator grades against the agreed contract
Sprint contracts complement (not replace) the static DoD in docs/MERGE_GATES.md.

## External Knowledge
When freshness matters (library APIs, platform behavior, security advisories):
prefer sanctioned retrieval (web search, docs MCP, docs/references/)
over guessing from training data.
```

### 2. .claude/settings.json — Hooks & Permissions

```json
{
  "permissions": {
    "deny": [
      "Read(**/.env)",
      "Read(**/.env.*)",
      "Read(**/secrets/**)",
      "Read(**/*.key)",
      "Bash(gh pr merge:*)",
      "Bash(git push origin main:*)",
      "Bash(git merge main:*)",
      "Bash(rm -rf /:*)",
      "Bash(rm -rf ~:*)"
    ],
    "ask": [
      "Bash(curl:*)",
      "Bash(wget:*)",
      "Edit(src/auth/**)",
      "Edit(src/billing/**)",
      "Edit(infra/**)",
      "Edit(migrations/**)"
    ],

    "_ask_notes": "Protected paths (auth, billing, infra, migrations) are in ASK, not deny. This means: Claude Code prompts for human approval before each edit, and the PostToolUse hook flags the edit for security review. This is the 'high-friction reviewed edits' model — edits are possible but require explicit approval. If you want to fully block edits (no exceptions), move them to deny. Do not duplicate in both — deny always wins.",
    "allow": [
      "Bash(make:*)",
      "Bash(git status:*)",
      "Bash(git diff:*)",
      "Bash(git log:*)",
      "Bash(git add:*)",
      "Bash(git commit:*)",
      "Bash(grep:*)",
      "Bash(jq:*)",
      "Edit(docs/**)",
      "Edit(scripts/**)"
    ]
  },

  "_permissions_notes": "IMPORTANT: deny rules are evaluated first and always win. Protected paths (auth, billing, infra, migrations) are in ASK — human approval required before each edit, plus PostToolUse flags for security review. This is 'high-friction reviewed edits', not 'blocked'. Secrets and destructive commands remain in DENY (fully blocked). We intentionally do NOT include broad Edit(src/**) in allow because non-protected src/ paths use the default permission mode (Normal mode prompts for approval). This follows the playbook principle: prefer structural safety over permissive defaults.",

  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "scripts/hooks/session-start.sh",
            "timeout": 10
          }
        ]
      }
    ],

    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "scripts/hooks/pre-tool-use-guard.sh",
            "timeout": 5
          }
        ]
      }
    ],

    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "scripts/hooks/post-edit-validate.sh",
            "timeout": 30
          }
        ]
      }
    ],

    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "scripts/hooks/stop-gate.sh",
            "timeout": 15
          }
        ]
      }
    ]
  }
}
```

### 3. Hook Scripts

> **Portability principle:** Any data used for enforcement (hashes, UUIDs, timestamps)
> must be produced by a single portable primitive, shared across all hooks and subagents.
> Two "equivalent" implementations can produce subtle mismatches. All hooks source
> `scripts/lib/portable-hash.sh` and `scripts/lib/portable-uuid.sh` instead of
> inlining their own logic.

#### `scripts/lib/portable-hash.sh`

```bash
#!/usr/bin/env bash
# Single source of truth for portable hashing across macOS and Linux.
# Used by: stop-gate.sh, verifier subagent, canary runner.
# MUST be sourced, not executed: source scripts/lib/portable-hash.sh

portable_sha256() {
  # Reads stdin, outputs hex digest only (no filename)
  if command -v sha256sum >/dev/null 2>&1; then
    sha256sum | cut -d' ' -f1
  elif command -v shasum >/dev/null 2>&1; then
    shasum -a 256 | cut -d' ' -f1
  elif command -v python3 >/dev/null 2>&1; then
    python3 -c "import sys,hashlib; print(hashlib.sha256(sys.stdin.buffer.read()).hexdigest())"
  else
    echo "ERROR: no sha256 tool available" >&2
    echo "HASH_UNAVAILABLE"
  fi
}

# Convenience: compute dirty-files hash for current worktree
dirty_files_hash() {
  git diff --name-only 2>/dev/null | sort | portable_sha256
}
```

#### `scripts/lib/portable-uuid.sh`

```bash
#!/usr/bin/env bash
# Single source of truth for portable UUID generation across macOS and Linux.
# Used by: session-start.sh, canary runner.
# MUST be sourced, not executed: source scripts/lib/portable-uuid.sh

portable_uuid() {
  uuidgen 2>/dev/null \
    || cat /proc/sys/kernel/random/uuid 2>/dev/null \
    || python3 -c "import uuid; print(uuid.uuid4())" 2>/dev/null \
    || echo "run-$(date +%s)-$$"
}
```

#### `scripts/hooks/session-start.sh`

```bash
#!/usr/bin/env bash
# SessionStart hook — loads harness context, generates run_id, starts trace
# Runtime state is TASK-SCOPED (by branch name) to prevent concurrent session conflicts.
set -euo pipefail
source "$(dirname "$0")/../lib/portable-uuid.sh"

HARNESS_META=".harness/harness.meta.json"

# --- Derive task scope from branch name ---
# Each task operates on its own branch; runtime state is scoped accordingly.
# This prevents concurrent sessions on different branches from conflicting.
TASK_ID=$(git branch --show-current 2>/dev/null | sed 's|.*/||' || echo "default")
TASK_DIR=".claude/tasks/${TASK_ID}"
mkdir -p "${TASK_DIR}"

PROGRESS="${TASK_DIR}/progress.json"
# Fall back to root progress.json for backward compat
[ ! -f "$PROGRESS" ] && [ -f "progress.json" ] && PROGRESS="progress.json"

# --- Generate run_id using portable helper ---
RUN_ID=$(portable_uuid)
echo "$RUN_ID" > "${TASK_DIR}/current-run-id"
export HARNESS_RUN_ID="$RUN_ID"
export HARNESS_TASK_DIR="$TASK_DIR"

# --- Create trace file (task-scoped) ---
mkdir -p .claude/traces
TRACE_FILE=".claude/traces/run-${RUN_ID}.jsonl"

OUTPUT=""

# Check harness version
if [ -f "$HARNESS_META" ]; then
  VERSION=$(jq -r '.playbook_version // "unknown"' "$HARNESS_META")
  LAST_AUDIT=$(jq -r '.last_audit // "never"' "$HARNESS_META")
  OUTPUT+="Harness v${VERSION} (last audit: ${LAST_AUDIT}). "
else
  OUTPUT+="WARNING: No harness.meta.json found — run bootstrap. "
fi

# Check progress state
if [ -f "$PROGRESS" ]; then
  IN_PROGRESS=$(jq '[.features[] | select(.status == "in_progress")] | length' "$PROGRESS" 2>/dev/null || echo "0")
  PENDING=$(jq '[.features[] | select(.status == "pending")] | length' "$PROGRESS" 2>/dev/null || echo "0")
  COMPLETED=$(jq '[.features[] | select(.status == "completed")] | length' "$PROGRESS" 2>/dev/null || echo "0")
  OUTPUT+="Progress: ${COMPLETED} done, ${IN_PROGRESS} in-progress, ${PENDING} pending. "
else
  OUTPUT+="No progress.json found. "
fi

# Check init.sh
if [ ! -f "init.sh" ]; then
  OUTPUT+="WARNING: No init.sh — consider running the initializer agent. "
fi

OUTPUT+="Run ID: ${RUN_ID}. "

# --- Increment dream session counter ---
DREAM_STATE=".claude/memory/dream-state.json"
if [ -f "$DREAM_STATE" ]; then
  SESSIONS=$(jq -r '.sessions_since_last // 0' "$DREAM_STATE" 2>/dev/null || echo 0)
  SESSIONS=$((SESSIONS + 1))
  jq ".sessions_since_last = $SESSIONS" "$DREAM_STATE" > "${DREAM_STATE}.tmp" && mv "${DREAM_STATE}.tmp" "$DREAM_STATE"
else
  mkdir -p .claude/memory
  echo '{"last_consolidation":"never","sessions_since_last":1,"total_consolidations":0}' > "$DREAM_STATE"
fi

# --- Write first trace event ---
echo '{"ts":'$(date +%s)',"event":"session.start","run_id":"'$RUN_ID'","agent":"main","head_sha":"'$(git rev-parse HEAD 2>/dev/null || echo unknown)'","context":"'"${OUTPUT}"'"}' >> "$TRACE_FILE"

# Output using hookSpecificOutput (current Anthropic format)
cat <<EOF
{
  "hookSpecificOutput": {
    "hookEventName": "SessionStart",
    "additionalContext": "${OUTPUT}"
  }
}
EOF
```

#### `scripts/hooks/pre-tool-use-guard.sh`

```bash
#!/usr/bin/env bash
# PreToolUse hook — blocks dangerous Bash commands, emits failure codes
set -euo pipefail

INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // ""')
# Task-scoped run_id (prevents concurrent session conflicts)
TASK_ID=$(git branch --show-current 2>/dev/null | sed 's|.*/||' || echo "default")
RUN_ID=$(cat ".claude/tasks/${TASK_ID}/current-run-id" 2>/dev/null || echo "${HARNESS_RUN_ID:-unknown}")
TRACE_FILE=".claude/traces/run-${RUN_ID}.jsonl"

# Block broad file dumps (Librarian pattern enforcement)
if echo "$COMMAND" | grep -qE 'cat \*\*/|find .* -exec cat'; then
  echo '{"ts":'$(date +%s)',"event":"tool.denied","run_id":"'$RUN_ID'","agent":"main","tool":"Bash","command":"'"$(echo $COMMAND | head -c 80)"'","failure_code":"TOOL_DENIED_BROAD_DUMP"}' >> "$TRACE_FILE" 2>/dev/null
  cat <<'EOF' >&2
{"hookSpecificOutput":{"hookEventName":"PreToolUse","permissionDecision":"deny","permissionDecisionReason":"TOOL_DENIED_BROAD_DUMP: Blocked broad file dump. Use make project-summary or make find-refs SYMBOL=x instead."}}
EOF
  exit 2
fi

# Block direct merge/deploy
if echo "$COMMAND" | grep -qE 'gh pr merge|git push origin main|git merge main'; then
  echo '{"ts":'$(date +%s)',"event":"tool.denied","run_id":"'$RUN_ID'","agent":"main","tool":"Bash","command":"'"$(echo $COMMAND | head -c 80)"'","failure_code":"TOOL_DENIED_MERGE"}' >> "$TRACE_FILE" 2>/dev/null
  cat <<'EOF' >&2
{"hookSpecificOutput":{"hookEventName":"PreToolUse","permissionDecision":"deny","permissionDecisionReason":"TOOL_DENIED_MERGE: Blocked direct merge. Use the merge-checker subagent."}}
EOF
  exit 2
fi

# Block rm -rf on non-worktree paths
if echo "$COMMAND" | grep -qE 'rm -rf /|rm -rf ~'; then
  echo '{"ts":'$(date +%s)',"event":"tool.denied","run_id":"'$RUN_ID'","agent":"main","tool":"Bash","command":"'"$(echo $COMMAND | head -c 80)"'","failure_code":"TOOL_DENIED_DESTRUCTIVE"}' >> "$TRACE_FILE" 2>/dev/null
  cat <<'EOF' >&2
{"hookSpecificOutput":{"hookEventName":"PreToolUse","permissionDecision":"deny","permissionDecisionReason":"TOOL_DENIED_DESTRUCTIVE: Blocked dangerous rm -rf outside worktree."}}
EOF
  exit 2
fi

exit 0
```

#### `scripts/hooks/post-edit-validate.sh`

```bash
#!/usr/bin/env bash
# PostToolUse hook — triggers validation after code edits
set -euo pipefail

INPUT=$(cat)
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // ""')

FEEDBACK=""

# --- Staleness header check for docs ---
if echo "$FILE_PATH" | grep -q "^docs/"; then
  if ! grep -q "Last verified:" "$FILE_PATH" 2>/dev/null; then
    FEEDBACK+="Doc edited without staleness header. Add: <!-- Last verified: $(date +%Y-%m-%d) | Commit: $(git rev-parse --short HEAD) | Status: CURRENT --> "
  fi
fi

# --- Protected path flagging ---
if echo "$FILE_PATH" | grep -qE "^src/(auth|billing)/|^infra/|^migrations/"; then
  FEEDBACK+="Protected path edited: ${FILE_PATH}. This PR will require human approval before merge. "
fi

# --- Scoped lint check (repo-contract-dependent: only works if make lint
#     supports file-scoped invocation; adapt to your repo's lint contract) ---
# NOTE: commands inside if/else are safe under set -e (the if catches non-zero)
if echo "$FILE_PATH" | grep -qE '\.(ts|tsx|js|jsx|py|rs)$'; then
  if make lint -- "$FILE_PATH" >/dev/null 2>&1; then
    : # lint passed
  else
    LINT_EXIT=$?
    if [ $LINT_EXIT -eq 2 ]; then
      # Convention: exit 2 = file-scoped lint not supported by this repo
      FEEDBACK+="This repo may not support file-scoped lint. Run make lint manually. "
    else
      FEEDBACK+="Lint issues detected in ${FILE_PATH}. Run make lint to see details. "
    fi
  fi
fi

# --- Scoped type check for TypeScript (only if npx/tsc available) ---
if echo "$FILE_PATH" | grep -qE '\.(ts|tsx)$'; then
  if command -v npx >/dev/null 2>&1; then
    if npx tsc --noEmit --pretty false "$FILE_PATH" >/dev/null 2>&1; then
      : # type check passed
    else
      FEEDBACK+="Type errors in ${FILE_PATH}. Fix before continuing. "
    fi
  fi
fi

if [ -n "$FEEDBACK" ]; then
  cat <<EOF
{"hookSpecificOutput":{"hookEventName":"PostToolUse","additionalContext":"${FEEDBACK}"}}
EOF
fi

exit 0
```

#### `scripts/hooks/stop-gate.sh`

```bash
#!/usr/bin/env bash
# Stop hook — FAST gatekeeper for Definition of Done
# Design: this hook checks only fast, local conditions.
# Heavy validation is delegated to the verifier subagent.
# CRITICAL: verification is bound to git state, not time.
# If files change after verification, the evidence bundle becomes invalid.
set -euo pipefail
source "$(dirname "$0")/../lib/portable-hash.sh"

# Task-scoped runtime state (prevents concurrent session conflicts)
TASK_ID=$(git branch --show-current 2>/dev/null | sed 's|.*/||' || echo "default")
TASK_DIR=".claude/tasks/${TASK_ID}"
RUN_ID=$(cat "${TASK_DIR}/current-run-id" 2>/dev/null || echo "${HARNESS_RUN_ID:-unknown}")
TRACE_FILE=".claude/traces/run-${RUN_ID}.jsonl"
PROGRESS="${TASK_DIR}/progress.json"
[ ! -f "$PROGRESS" ] && PROGRESS="progress.json"
ISSUES=""
FAILURE_CODES=""

# --- Fast check 1: progress.json has no orphaned in_progress ---
if [ -f "$PROGRESS" ]; then
  IN_PROGRESS=$(jq '[.features[] | select(.status == "in_progress")] | length' "$PROGRESS" 2>/dev/null || echo "0")
  if [ "$IN_PROGRESS" -gt 0 ]; then
    ISSUES+="PROGRESS_ORPHANED: features still in_progress. "
    FAILURE_CODES+="PROGRESS_ORPHANED "
  fi

  UNVERIFIED=$(jq '[.features[] | select(.status == "completed" and (.verified_by == null or .verified_by == ""))] | length' "$PROGRESS" 2>/dev/null || echo "0")
  if [ "$UNVERIFIED" -gt 0 ]; then
    ISSUES+="PROGRESS_UNVERIFIED: completed features missing verified_by. "
    FAILURE_CODES+="PROGRESS_UNVERIFIED "
  fi
else
  :
fi

# --- Fast check 2: verifier evidence bundle is STATE-BOUND (task-scoped) ---
VERIFIER_STAMP="${TASK_DIR}/evidence-bundle.json"
if [ -f "$VERIFIER_STAMP" ]; then
  STAMP_RESULT=$(jq -r '.result // ""' "$VERIFIER_STAMP" 2>/dev/null)
  if [ "$STAMP_RESULT" != "pass" ]; then
    ISSUES+="VERIFY_NOT_INVOKED: evidence bundle result is not 'pass'. "
    FAILURE_CODES+="VERIFY_NOT_INVOKED "
  else
    STAMP_HEAD=$(jq -r '.correlation["git.head_sha"] // .head_sha // ""' "$VERIFIER_STAMP" 2>/dev/null)
    STAMP_DIRTY=$(jq -r '.correlation.dirty_files_hash // .dirty_files_hash // ""' "$VERIFIER_STAMP" 2>/dev/null)
    CURRENT_HEAD=$(git rev-parse HEAD 2>/dev/null || echo "unknown")
    CURRENT_DIRTY=$(dirty_files_hash)

    if [ "$STAMP_HEAD" != "$CURRENT_HEAD" ]; then
      ISSUES+="VERIFY_HEAD_MISMATCH: commits made after verification. "
      FAILURE_CODES+="VERIFY_HEAD_MISMATCH "
    elif [ "$STAMP_DIRTY" != "$CURRENT_DIRTY" ]; then
      # Dirty hash diverged — but is the change in verified files or unrelated cleanup?
      # Get files that changed since verification
      CHANGED_SINCE=$(git diff --name-only 2>/dev/null | sort)
      # Get files that were verified (from evidence bundle)
      VERIFIED_FILES=$(jq -r '.touched_files[]? // empty' "$VERIFIER_STAMP" 2>/dev/null | sort)
      # Check overlap: did any verified file change?
      OVERLAP=$(comm -12 <(echo "$CHANGED_SINCE") <(echo "$VERIFIED_FILES") 2>/dev/null)
      if [ -n "$OVERLAP" ]; then
        ISSUES+="VERIFY_DIRTY_HASH_MISMATCH: verified files modified after verification: ${OVERLAP}. "
        FAILURE_CODES+="VERIFY_DIRTY_HASH_MISMATCH "
      else
        # Changes are to non-verified files (trivial cleanup) — warn but allow
        echo '{"ts":'$(date +%s)',"event":"stop.warn","run_id":"'$RUN_ID'","agent":"main","warning":"DIRTY_HASH_TRIVIAL: post-verify changes to non-verified files only"}' >> "$TRACE_FILE" 2>/dev/null
      fi
    fi
  fi
else
  ISSUES+="VERIFY_NOT_INVOKED: verifier subagent was not run. "
  FAILURE_CODES+="VERIFY_NOT_INVOKED "
fi

# --- Decision ---
if [ -n "$ISSUES" ]; then
  echo '{"ts":'$(date +%s)',"event":"stop.blocked","run_id":"'$RUN_ID'","agent":"main","failure_codes":"'"${FAILURE_CODES}"'"}' >> "$TRACE_FILE" 2>/dev/null
  cat <<EOF
{"continue": false, "stopReason": "${ISSUES}"}
EOF
  exit 0
fi

# --- Success: emit trace event ---
echo '{"ts":'$(date +%s)',"event":"stop.allowed","run_id":"'$RUN_ID'","agent":"main","state_bound":"verified"}' >> "$TRACE_FILE" 2>/dev/null

exit 0
```

> **Design rationale — Stop hook architecture (3 layers):**
>
> **1. What the Stop hook verifies (fast, < 1s):**
> - Presence of a valid evidence bundle (`.claude/tasks/<task-id>/evidence-bundle.json`)
>   with `result: "pass"` and all check statuses `"pass"` or `"n/a"`
> - State binding: the bundle's `git.head_sha` must match current HEAD (hard block
>   on new commits). For `dirty_files_hash` divergence, the hook checks whether
>   the changed files overlap with the verified file set from the evidence bundle.
>   If only non-verified files changed (trivial cleanup like removing a console.log),
>   the hook warns but allows the stop. If verified files changed, it blocks.
>   This prevents the brittleness of re-running the entire verification suite
>   over non-breaking post-verification cleanup.
> - Session invariants: no orphaned `in_progress` features, no completed
>   features missing `verified_by` in progress.json
>
> **2. What the Stop hook does NOT verify:**
> - It does not run `make test`, `make lint`, or `make test-stress`
> - It does not check proof artifact contents
> - It does not produce evidence — only validates evidence already produced
>
> **3. Why this split:**
> - Anthropic's hooks run synchronously; heavy checks (minutes) would make
>   the hook expensive and brittle
> - The **verifier subagent** produces the evidence bundle (correlation IDs,
>   per-check results, touched files, failure codes, git state)
> - The **stop hook** validates the bundle's integrity against current state
> - This separation means: verifier = evidence producer, stop hook = evidence auditor
>
> For Stop/SubagentStop, this blueprint uses `continue: false` + `stopReason`
> rather than `decision: "block"`, because Anthropic's docs state that
> `continue: false` takes precedence for Stop hooks.
>
> **Terminology:** This blueprint uses "evidence bundle" rather than "stamp"
> because the artifact contains structured correlation IDs, per-check results,
> touched file lists, and failure codes — not just a binary pass/fail marker.
>
> **Artifact note:** Runtime state is task-scoped under `.claude/tasks/<task-id>/`
> to prevent concurrent session conflicts. Each task (identified by git branch name)
> gets its own `current-run-id`, `progress.json`, and `evidence-bundle.json`.
> These are harness-local artifacts, not Claude Code built-ins. Similarly, `init.sh`
> and `.claude/traces/` are harness artifacts. Claude Code's native surfaces are
> CLAUDE.md, settings.json, agents, and commands; everything else is repo-local
> harness state that Claude Code uses but does not manage.

> **settings.local.json caution:** Anthropic documents `.claude/settings.local.json`
> as local project settings that override shared project settings in precedence.
> This means individual developers can weaken project-level deny rules or hooks.
> **Never rely on shared project settings alone for critical safety boundaries if
> local overrides are allowed.** Use managed settings
> (`/Library/Application Support/ClaudeCode/managed-settings.json` on macOS) for
> critical deny rules, or add CI-level backstops that verify harness compliance
> independently of Claude Code settings.

> **Enforcement hierarchy: CI is canonical, hooks are local accelerators.**
> Claude Code hooks and permissions are local enforcement — they run on the
> developer's machine and can be bypassed (settings.local.json, bypass mode,
> or simply not running Claude Code). They are guardrails, not walls.
>
> **Sandbox isolation is the strongest safety primitive.** Anthropic documents
> Seatbelt (macOS) and bubblewrap (Linux) as native sandboxing for Bash
> commands. For agent runs, prefer worktree isolation (one worktree per task,
> isolated filesystem + network) over permission rules alone. Permission rules
> are defense in depth; sandboxing is the structural barrier.
>
> **CI is the canonical enforcer.** Every critical rule in this blueprint should
> ALSO be enforced in CI, independently of Claude Code:
>
> | Rule | Claude Code surface (local) | CI backstop (canonical) |
> |---|---|---|
> | Architecture boundaries | CLAUDE.md instruction | `dependency-cruiser` / `import-linter` in CI |
> | Protected-path approvals | settings.json deny rules | CODEOWNERS or branch protection rules |
> | Definition of Done | Stop hook + verifier subagent | CI pipeline checks before merge |
> | Golden principles | CLAUDE.md + golden-sweep agent | Lint scripts in CI |
> | Secrets detection | settings.json Read deny rules | `make scan-secrets` in CI |
> | Test determinism | `make test-stress` via verifier | `make test-stress` in CI pipeline |
>
> The hooks make the developer experience faster (catch issues before CI).
> CI makes the enforcement authoritative (catch issues the hooks missed).
> Neither alone is sufficient. Both together form the real harness.

### 4. Project Subagents

#### `.claude/agents/initializer.md`

```markdown
---
name: initializer
description: Use for FIRST SESSION on a new task or repo. Sets up environment, creates progress.json, runs init.sh, and makes the initial git commit. Use proactively when no progress.json exists.
tools: Read, Write, Edit, Bash, Glob, Grep
model: sonnet
---

You are the Initializer agent. Your job is to set up the project environment
for long-running agent work, following the Initializer pattern in docs/INITIALIZER.md.

## Your outputs (all required):

1. **init.sh** — bootstrap script that installs deps and boots the dev environment.
   Must be idempotent (safe to re-run).

2. **progress.json** — structured state file with:
   - features[]: { name, status: "pending", notes: "", verified_by: "" }
   - last_session: { date, agent_id, summary, next_steps }
   - known_issues: []
   - environment: { bootstrap_script, install_status, startup_status, smoke_test_status, ... }
   Status values: exactly "pending", "in_progress", "completed", "blocked".

3. **Initial git commit** — clean baseline for rollback.

4. **Environment verification** — run init.sh, then verify:
   - Install succeeded
   - App starts (make dev)
   - Tests pass (make test)
   - Populate the environment block in progress.json with results.
   If any step fails, mark it "fail" or "partial" and note the issue.

## Rules:
- Analyze the existing codebase before creating anything.
- Do not hallucinate features — only list what the task/project actually requires.
- All features start as "pending".
- Make one clean git commit after setup is complete.
- Return a concise summary of what was set up and any issues found.
```

#### `.claude/agents/verifier.md`

```markdown
---
name: verifier
description: Use proactively BEFORE completing a task or ending a session. Checks that the Definition of Done is met, proof artifacts exist, and progress.json is up to date. MUST be invoked before session end — the Stop hook will block if it was not.
tools: Read, Glob, Grep, Bash
model: sonnet
---

You are the Verifier agent. Check that the current work meets
the Definition of Done defined in docs/MERGE_GATES.md.

## Checklist (all must pass):
1. `make test` passes
2. `make lint` passes
3. Proof artifacts exist in proof/<issue-id>/ (appropriate to task class)
4. Related docs updated if they became stale (check staleness headers)
5. Protected-path gates satisfied if applicable
6. ExecPlan updated if one was used
7. `make test-stress` passes on touched test files
8. progress.json updated with current state

## Output:
Return a structured pass/fail report with failure codes from docs/OBSERVABILITY.md.
For each failing item, include the failure code and what the agent should do to fix it.

## On full pass:
If ALL checks pass, write the harness-local evidence bundle.
First, source the portable hash helper for cross-platform compatibility:

    source scripts/lib/portable-hash.sh
    TASK_ID=$(git branch --show-current | sed 's|.*/||')
    TASK_DIR=".claude/tasks/${TASK_ID}"
    mkdir -p "${TASK_DIR}"
    RUN_ID=$(cat "${TASK_DIR}/current-run-id" 2>/dev/null || echo "${HARNESS_RUN_ID:-unknown}")
    HEAD_SHA=$(git rev-parse HEAD)
    BRANCH=$(git branch --show-current)
    DIRTY_HASH=$(dirty_files_hash)
    TOUCHED=$(git diff --name-only main... | jq -R . | jq -s .)

    cat > "${TASK_DIR}/evidence-bundle.json" <<BUNDLE
    {
      "correlation": {
        "run_id": "$RUN_ID",
        "task_id": "$TASK_ID",
        "git.head_sha": "$HEAD_SHA",
        "git.branch": "$BRANCH",
        "dirty_files_hash": "$DIRTY_HASH"
      },
      "result": "pass",
      "checks": { ... fill per-check results ... },
      "touched_files": $TOUCHED,
      "failure_codes": [],
      "timestamp_epoch": $(date +%s)
    }
    BUNDLE

The bundle is STATE-BOUND: the Stop hook verifies head_sha and dirty_files_hash.
If anything changes after verification, the stop is rejected.

Also append a trace event:
    echo '{"ts":'$(date +%s)',"event":"verifier.pass","run_id":"'$RUN_ID'","agent":"verifier","checks":8,"head_sha":"'$HEAD_SHA'"}' >> .claude/traces/run-${RUN_ID}.jsonl

Do NOT fix things yourself — report only. The main agent fixes.
```

#### `.claude/agents/docs-maintainer.md`

```markdown
---
name: docs-maintainer
description: Scans docs/ for stale documentation that doesn't match current code behaviour. Opens targeted fix-up suggestions. Use periodically or when docs staleness is suspected.
tools: Read, Glob, Grep, Bash
model: sonnet
---

You are the Documentation Maintainer agent.

## Your job:
1. Scan all files in docs/ for staleness headers.
2. For each doc, check if "Last verified" date exceeds its class SLA:
   - Architecture/security/reliability: 90 days
   - Product specs/active plans: 30 days
   - Generated docs: compare source commit vs doc commit
3. For stale docs, check if the content still matches actual code.
4. Report findings as a structured list: which docs are stale,
   what specifically is wrong, and a suggested fix.
5. Do NOT invent content — only correct based on what code actually does.

## Output:
Return a summary of findings. If no docs are stale, say so.
```

#### `.claude/agents/merge-checker.md`

```markdown
---
name: merge-checker
description: Runs all merge gates from docs/MERGE_GATES.md before landing a PR. Use instead of direct gh pr merge or git merge commands.
tools: Read, Glob, Grep, Bash
model: sonnet
---

You are the Merge Gate Checker. Execute all gates defined in
docs/MERGE_GATES.md and report pass/fail for each.

## Gate sequence:
1. CI green on final commit (run make test && make lint)
2. No secrets detected (run make scan-secrets or grep for patterns)
3. Linear commit history (check for merge commits)
4. Complexity density report (run scripts/complexity-check.sh — soft warning)
5. Stress test on touched tests (run make test-stress)
6. Proof artifacts exist in proof/<issue-id>/
7. Protected paths check — if modified, human approval label required
8. Definition of Done — all items met

## Output:
Return a gate-by-gate pass/fail report.
If ALL gates pass, output: "All gates passed. Safe to land."
If ANY gate fails, output which failed and why. Do NOT merge.
```

#### `.claude/agents/security-reviewer.md`

```markdown
---
name: security-reviewer
description: Reviews changes to protected paths (src/auth/**, src/billing/**, infra/**, migrations/**) for security implications. Use proactively when protected paths are modified.
tools: Read, Glob, Grep
model: sonnet
---

You are the Security Reviewer agent. You have READ-ONLY access.

## Your job:
1. Review the diff in protected paths for:
   - Credential handling changes
   - Authentication/authorization logic changes
   - Data validation at boundaries
   - New dependencies or external calls
   - Permission scope changes
2. Flag anything that looks risky with a clear explanation.
3. Do NOT make changes — report findings only.

## Output:
Structured security review with risk level (low/medium/high) per finding.
If no issues found, say "No security concerns identified."
```

#### `.claude/agents/lessons-sweep.md`

```markdown
---
name: lessons-sweep
description: Reviews accumulated lessons in docs/generated/lessons.md and proposes harness upgrades. Use weekly or when 5+ unpromoted lessons have accumulated. Promotes patterns into golden principles, hooks, verifier rules, or command contracts.
tools: Read, Glob, Grep, Bash
model: sonnet
---

You are the Lessons Sweep agent. Your job is to close the feedback loop
between agent mistakes and harness improvements.

## Your process:
1. Read docs/generated/lessons.md
2. Count unpromoted lessons (those without "Promoted →" markers)
3. If fewer than 5 unpromoted lessons: report "Not enough lessons to promote" and stop.
4. If 5 or more: group by theme:
   - Repeated pattern mistakes → propose a new golden principle for docs/GOLDEN_PRINCIPLES.md
   - Repeated safety issues   → propose a new permission rule or hook for .claude/settings.json
   - Repeated verification gaps → propose a new check for .claude/agents/verifier.md
   - Repeated tooling gaps     → propose a new make target or command contract

## Output:
For each proposed upgrade, return:
  - The lesson(s) that motivated it
  - The exact file to modify
  - The proposed change (as a diff or new content)
  - Which lessons to mark as "Promoted → <artifact name>"

Do NOT apply changes yourself — return the proposals for the main agent to implement.
The main agent will then run canary tasks to verify the harness change doesn't regress.
```

#### `.claude/agents/dream-sweep.md`

```markdown
---
name: dream-sweep
description: Memory consolidation agent (Auto Dream pattern). Runs the 4-phase dream cycle to consolidate agent memory. Use between sessions or when memory has accumulated from 5+ sessions. Compatible with Claude Code Auto Dream when it ships natively.
tools: Read, Write, Edit, Bash, Glob, Grep
model: sonnet
---

You are performing a dream — a reflective pass over your memory files.
Synthesize what you've learned recently into durable, well-organized
memories so that future sessions can orient quickly.

Memory directory: .claude/memory/
Session transcripts: .claude/traces/ (JSONL files — grep narrowly, don't read whole files)

## Phase 1 — Orient
- ls the memory directory to see what already exists
- Read MEMORY.md to understand the current index
- Skim existing topic files so you improve them rather than creating duplicates

## Phase 2 — Gather recent signal
Look for new information worth persisting. Priority sources:
1. Recent trace files (.claude/traces/run-*.jsonl) — grep for corrections,
   decisions, patterns, errors
2. Existing memories that drifted — facts that contradict current codebase
3. docs/generated/lessons.md — unpromoted lessons that should become memories

Don't exhaustively read traces. Grep narrowly for suspected patterns:
  grep -rn "<term>" .claude/traces/ --include="*.jsonl" | tail -50

## Phase 3 — Consolidate
For each signal worth persisting, write or update a topic file:
- MERGE new signal into existing topic files (don't create near-duplicates)
- CONVERT relative dates to absolute dates ("yesterday" → "2026-03-30")
- DELETE contradicted facts at the source
- REMOVE stale memories (debugging notes about deleted files)
- MERGE overlapping entries (3 sessions noted same thing → one clean entry)

## Phase 4 — Prune and index
Update MEMORY.md so it stays UNDER 200 LINES.
It is an INDEX, not a dump:
- One-line description per topic file, linked
- Remove pointers to stale/superseded topic files
- Demote verbose entries: gist in index, detail in topic file
- Add pointers to newly important memories
- Resolve contradictions: if two files disagree, fix the wrong one

## Dual gate check
Before doing anything, read .claude/memory/dream-state.json.
If BOTH conditions are not met, report "Not yet time to dream" and stop:
  1. ≥ 24 hours since last consolidation
  2. ≥ 5 sessions since last consolidation

After completing the cycle, update dream-state.json:
  {"last_consolidation": "<now ISO>", "sessions_since_last": 0, "total_consolidations": <n+1>}

## Output
Return a brief summary of what you consolidated, updated, or pruned.
If nothing changed (memories are already tight), say so.

## Native compatibility
When Claude Code Auto Dream ships natively, this subagent becomes
redundant for Claude Code users. The memory directory (.claude/memory/)
and index format (MEMORY.md < 200 lines) are intentionally aligned
with the native mechanism. For non-Claude tools, this subagent
continues to provide the same consolidation capability.
```

#### `.claude/agents/planner.md`

```markdown
---
name: planner
description: Use for greenfield projects or new feature suites. Expands a short product prompt (1-4 sentences) into a full product spec with features, sprints, design language, and AI integration opportunities. Distinct from the initializer (which sets up the environment).
tools: Read, Write, Edit, Bash, Glob, Grep
model: sonnet
---

You are the Planner agent. Your job is to expand a short product concept
into a structured, ambitious product specification.

## Your process:
1. Read the human's prompt (1-4 sentences)
2. Analyze the existing codebase if one exists (make project-summary)
3. Expand into a full spec covering:
   - Feature list organized into sprints/phases
   - High-level technical design (stack, data model, API shape)
   - Design language / visual identity guidelines
   - AI integration opportunities (where Claude can add value in the app)

## Rules:
- Be AMBITIOUS about scope — expand well beyond the literal prompt
- Stay at PRODUCT level — do not specify granular implementation details
  (errors in detailed specs cascade into broken implementations)
- Constrain DELIVERABLES, let the generator figure out the PATH
- If a frontend design skill is available, read it and incorporate its
  principles into the design language section

## Output:
Write the spec to .claude/handoffs/spec.md
Also update progress.json with the feature list (all "pending").
Return a summary of what was planned.
```

#### `.claude/agents/evaluator.md`

```markdown
---
name: evaluator
description: Use after each sprint or feature completion to actively exercise the running application, grade against negotiated criteria, and produce actionable feedback. The generator-evaluator loop is the primary quality mechanism for complex builds. Distinct from the verifier (which checks DoD gates).
tools: Read, Glob, Grep, Bash
model: sonnet
---

You are the Evaluator agent. Your job is to actively exercise the running
application and produce skeptical, actionable feedback.

## Your process:
1. Read the sprint contract from .claude/handoffs/sprint-<n>-contract.md
2. ACTIVELY exercise the running application:
   - Navigate pages, click through UI flows
   - Call API endpoints with realistic data
   - Inspect database states
   - Use Playwright MCP, curl, or equivalent — NOT just code review
3. Grade each criterion in the contract
4. Write detailed feedback to .claude/handoffs/sprint-<n>-feedback.md

## Grading rules:
- Be SKEPTICAL. Your default stance is "this probably has issues."
- Test EDGE CASES, not just happy paths
- If a criterion falls below threshold → the sprint FAILS
- Include specific file:line references for each issue found
- For subjective quality (design, UX): use the grading criteria
  from the sprint contract or docs/QUALITY_CRITERIA.md

## Sprint contract negotiation:
When the generator proposes a sprint contract, REVIEW it:
- Are the criteria testable and specific enough?
- Does the proposal match the spec intent?
- Are edge cases covered?
If not, write feedback requesting changes in the contract file.

## Output format (.claude/handoffs/sprint-<n>-feedback.md):
  ## Sprint N Evaluation
  ### Overall: PASS / FAIL
  ### Criterion results:
  1. [criterion]: PASS/FAIL — [details, file:line refs]
  2. ...
  ### Bugs found:
  - [bug description, reproduction steps, file:line]
  ### Recommendations:
  - [what to fix, whether to refine or pivot]

Do NOT fix things yourself — grade and report only.
```

### 5. Slash Commands

#### `.claude/commands/audit-harness.md`

```markdown
Read .harness/PLAYBOOK_SKILL.md in full.
Then run PHASE 0 (audit only) from the bootstrap prompt.
Show the gap report comparing the current harness state
against the playbook. Do not modify anything.
```

#### `.claude/commands/init-task.md`

```markdown
Use the initializer agent to set up a new task: $ARGUMENTS

Create progress.json with the task features, run init.sh,
verify the environment, and make the initial git commit.
```

#### `.claude/commands/promote-lessons.md`

```markdown
Use the lessons-sweep subagent to review docs/generated/lessons.md.

If enough lessons have accumulated, review the proposed harness upgrades,
implement them, then run canary tasks to verify no regression.
Mark promoted lessons in lessons.md with a reference to the artifact they became.
```

---

## Implementation Sequence

### Pass 1 — Minimum Viable Claude Runtime
```
Generate:
  CLAUDE.md
  .claude/settings.json (permissions only, no hooks yet)
  .claude/agents/initializer.md
  .claude/handoffs/ (empty directory for inter-agent communication)
  .claude/memory/MEMORY.md (empty index with format instructions)
  .claude/memory/dream-state.json (initial trigger state)
  scripts/lib/portable-hash.sh
  scripts/lib/portable-uuid.sh
  progress.json
  init.sh

This gives you a functioning harness on day one.
```

### Pass 2 — Repo Docs & Command Contracts
```
Generate:
  docs/ARCHITECTURE.md
  docs/GOLDEN_PRINCIPLES.md
  docs/MERGE_GATES.md
  docs/SECURITY.md
  docs/TESTING.md
  docs/ROLLBACK.md
  docs/INITIALIZER.md
  docs/OBSERVABILITY.md (failure codes, correlation schema, evidence format)
  docs/CANARY_TASKS.md (canary task corpus — initially 5 representative tasks)
  docs/generated/lessons.md (empty template with format instructions)
  Makefile targets (project-summary, find-refs, check-docs, test-stress)
  scripts/complexity-check.sh
```

### Pass 3 — Hook Enforcement
```
Generate:
  scripts/hooks/session-start.sh (sources portable-uuid.sh, writes run_id file)
  scripts/hooks/pre-tool-use-guard.sh (sources run_id from file)
  scripts/hooks/post-edit-validate.sh (sources run_id from file)
  scripts/hooks/stop-gate.sh (sources portable-hash.sh, reads evidence bundle)
  Update .claude/settings.json with hook configuration

At this point the harness stops being advisory.
```

### Pass 4 — Subagents & Skills
```
Generate:
  .claude/agents/verifier.md
  .claude/agents/docs-maintainer.md
  .claude/agents/merge-checker.md
  .claude/agents/security-reviewer.md
  .claude/agents/golden-sweep.md
  .claude/agents/quality-scorer.md
  .claude/agents/lessons-sweep.md
  .claude/agents/dream-sweep.md (4-phase memory consolidation)
  .claude/agents/planner.md (for greenfield / new feature suites)
  .claude/agents/evaluator.md (adversarial QA / grading agent)
  .claude/skills/harness-audit/SKILL.md
  .claude/commands/audit-harness.md
  .claude/commands/init-task.md
  .claude/commands/promote-lessons.md
```

### Pass 5 — Verification, Canary Tasks & Observability
```
  Run the full canary task suite from docs/CANARY_TASKS.md.
  For each canary: record retries, failure codes, verifier results, time.
  Compare against baseline (or establish baseline on first run).
  Adjust hooks and permissions based on results.
  Verify traces are written to .claude/traces/.
  Verify make metrics-agent-behavior computes from trace data.
```

### Pass 6 — Codex Compatibility Mirror (OPTIONAL)
```
  Generate AGENTS.md from CLAUDE.md content.
  Verify both files stay in sync (add a make target or CI check).
```

> **Cross-tool reality check:** This blueprint is a Claude Code harness.
> The AGENTS.md mirror provides basic Codex compatibility, but the native
> surfaces (.claude/settings.json, hooks, subagents, commands, task-scoped
> state) have no Codex equivalent. If you operate both Claude Code and
> Codex seriously, maintain them as **sibling harnesses** sharing one
> conceptual playbook, not as one generated filesystem pretending to be
> equally native for both tools. The playbook is the shared theory layer;
> the blueprint and any future Codex-specific blueprint are the diverging
> concrete implementations.

---

## Agent Observability Contract

> **Implementation status:** This section defines the observability model for
> the Claude Code harness. Level 1 (correlation + failure codes + evidence bundle)
> is implementable now with hooks, subagents, and Make targets. Level 2
> (hierarchical spans + centralized trace store) requires an external runtime
> wrapper or orchestrator. Level 3 (real-time alerting, cross-session correlation)
> requires infrastructure investment. Start at Level 1; promote when workload justifies.
>
> **Academic basis:** This design draws on OpenTelemetry GenAI semantic conventions
> (development status), Reflexion/Self-Refine (structured feedback loops),
> SWE-Gym (trajectory-based verification), and traceability studies for SWE agents.

### 5.1 Correlation Schema

Every harness event must carry a minimum set of identifiers to enable
cross-event correlation. Without these, signals exist but cannot be chained.

```json
{
  "_comment": "Minimum correlation fields — present in every harness artifact",
  "correlation": {
    "run_id":          "uuid — unique per agent session/run",
    "task_id":         "string — issue/ticket ID driving this work",
    "session_id":      "string — Claude Code session identifier",
    "worktree_id":     "string — path to worktree if using isolated worktrees",
    "git.head_sha":    "string — HEAD at event time",
    "git.branch":      "string — current branch name",
    "agent.name":      "string — main | verifier | initializer | etc.",
    "policy.surface":  "string — hook | subagent | ci | manual"
  }
}
```

**Where these appear:**

| Artifact | Carries correlation? | Level |
|---|---|---|
| progress.json | run_id, task_id, git.head_sha | 1 (now) |
| .claude/tasks/<id>/evidence-bundle.json | full correlation + evidence | 1 (now) |
| Hook JSON outputs | run_id, failure_code, policy.surface | 1 (now) |
| docs/generated/lessons.md | task_id, run_id per lesson entry | 1 (now) |
| OTel spans (centralized) | full correlation as span attributes | 2 (target) |

**Implementation (Level 1):** Generate `run_id` in the SessionStart hook
and propagate it via environment variable `HARNESS_RUN_ID`. All subsequent
hooks and subagents read this variable and include it in their outputs.

### 5.2 Failure Code Taxonomy

Free-text failure reasons are readable but not queryable.
Standardize failure codes across all hooks, subagents, and CI gates.

```
FAILURE CODES (include in docs/OBSERVABILITY.md):

Verification failures:
  VERIFY_HEAD_MISMATCH         — commits made after verification
  VERIFY_DIRTY_HASH_MISMATCH   — uncommitted changes since verification
  VERIFY_NOT_INVOKED           — verifier subagent was never run
  VERIFY_TEST_FAIL             — make test failed during verification
  VERIFY_LINT_FAIL             — make lint failed during verification
  VERIFY_PROOF_MISSING         — proof artifacts not found
  VERIFY_DOC_STALE             — docs stale beyond SLA
  VERIFY_STRESS_FAIL           — make test-stress failed
  VERIFY_PLAN_NOT_UPDATED      — active exec-plan not updated

Permission / policy failures:
  TOOL_DENIED_MERGE            — direct merge command blocked
  TOOL_DENIED_BROAD_DUMP       — broad file dump blocked
  TOOL_DENIED_SECRET_READ      — .env / secrets read blocked
  TOOL_DENIED_DESTRUCTIVE      — rm -rf outside worktree blocked
  PROTECTED_PATH_EDIT          — edit to auth/billing/infra/migrations flagged

Workflow failures:
  LOOP_BREAKER_TRIGGERED       — 3 fix-and-rerun cycles exhausted
  PROGRESS_ORPHANED            — features still in_progress at stop
  PROGRESS_UNVERIFIED          — completed features missing verified_by

CI divergence:
  CI_BACKSTOP_FAIL             — CI caught issue that hooks missed
  CI_HOOK_DIVERGENCE           — hook passed but CI failed (or vice versa)
```

**Usage in hooks:** Every hook that blocks or warns should include a `failure_code`
in its JSON output alongside the human-readable reason:

```json
{"continue": false, "stopReason": "VERIFY_HEAD_MISMATCH: Commits made after verification."}
```

### 5.3 Evidence Bundle

The verifier subagent should produce a **structured evidence bundle**, not just
a pass/fail stamp. This makes verification auditable and replayable.

```json
{
  "_comment": "Evidence bundle — written by verifier subagent on full pass",
  "correlation": {
    "run_id": "...",
    "task_id": "...",
    "git.head_sha": "...",
    "git.branch": "...",
    "dirty_files_hash": "..."
  },
  "result": "pass",
  "checks": {
    "test":          { "status": "pass", "exit_code": 0, "duration_ms": 12340 },
    "lint":          { "status": "pass", "exit_code": 0, "duration_ms": 3200 },
    "proof":         { "status": "pass", "files": ["proof/ISSUE-42/screenshot.png"] },
    "docs_freshness":{ "status": "pass", "stale_count": 0 },
    "protected_paths":{ "status": "n/a", "reason": "no protected paths modified" },
    "exec_plan":     { "status": "pass", "plan": "docs/exec-plans/active/ISSUE-42.md" },
    "test_stress":   { "status": "pass", "runs": 10, "passed": 10, "profile": "unit" },
    "progress":      { "status": "pass", "completed": 3, "in_progress": 0 }
  },
  "touched_files": ["src/api/users.ts", "src/api/users.test.ts", "docs/ARCHITECTURE.md"],
  "failure_codes": [],
  "timestamp_epoch": 1742140800
}
```

**Stop hook integration:** The stop hook reads this bundle and verifies:
1. `result` is `"pass"`
2. `correlation.git.head_sha` matches current HEAD
3. `correlation.dirty_files_hash` matches current dirty state
4. All check statuses are `"pass"` or `"n/a"`

This replaces the simpler stamp with a richer, auditable artifact.

### 5.4 Trace vs Memory: Two Artifacts

Raw traces and operational memory serve different purposes.
Mixing them degrades both. Keep them separate.

| Artifact | Purpose | Mutable? | Retention |
|---|---|---|---|
| `.claude/traces/run-<id>.jsonl` | Immutable event log: every hook fire, tool call, denial, verification | No — append-only | Archive after task close |
| `docs/generated/lessons.md` | Condensed operational memory: dead-ends, invariants, failure patterns | Yes — promoted lessons get marked | Permanent (lessons are the audit trail for why rules exist) |
| `.claude/tasks/<id>/evidence-bundle.json` | Latest evidence bundle (task-scoped) | Overwritten per verification run | Current task only |
| `progress.json` | Structured task state | Yes — updated per feature | Current task lifecycle |

**Trace format (Level 1 — append to `.claude/traces/run-<id>.jsonl`):**

```jsonl
{"ts":1742140800,"event":"session.start","run_id":"...","agent":"main","context":"Harness v3.4.0, 2 pending features"}
{"ts":1742140812,"event":"tool.denied","run_id":"...","agent":"main","tool":"Bash","command":"cat **/*.ts","failure_code":"TOOL_DENIED_BROAD_DUMP"}
{"ts":1742141400,"event":"verifier.pass","run_id":"...","agent":"verifier","checks":8,"head_sha":"abc123"}
{"ts":1742141405,"event":"stop.allowed","run_id":"...","agent":"main","state_bound":"verified"}
```

**Implementation (Level 1):** Each hook appends one JSONL line to the trace file.
The SessionStart hook creates the file with the run_id. No external infra needed.

**Level 2 (target):** Replace JSONL files with OTel spans emitted to a collector.
The correlation schema maps directly to span attributes.

### 5.5 Derived Evaluations

Traces are not useful until they feed evaluations.
Define `make metrics-*` targets that compute from trace data, not from manual input.

```
METRICS DERIVED FROM TRACES (in Makefile):

make metrics-throughput
  Source: traces + git log
  Computes: time-to-merge, PRs/day, iteration count per task

make metrics-quality
  Source: traces + CI logs
  Computes: CI pass rate, defect escape rate, rollback frequency

make metrics-harness
  Source: traces + docs headers
  Computes: doc freshness score, boundary violations, flake rate,
            hook-vs-CI divergence rate

make metrics-safety
  Source: traces
  Computes: blocked egress count, secret-scan hits, permission denials,
            protected-path edits flagged vs approved

make metrics-agent-behavior
  Source: traces
  Computes: loop-breaker trigger rate, edits-after-verification rate,
            avg retries per task, human escalation rate, failure code
            distribution, verifier pass rate by task class
```

**Key derived evaluations:**

| Metric | Source | Alert threshold |
|---|---|---|
| Edits after verification | trace: edit events after verifier.pass | > 0 per run = investigate |
| Loop breaker rate | trace: LOOP_BREAKER_TRIGGERED / total runs | > 15% = harness too strict or agent too weak |
| Hook-CI divergence | trace: hook passed + CI failed (or reverse) | > 5% = hooks out of sync |
| Verification rework | trace: verifier.fail → fix → verifier.pass count | > 3 per run = DoD unclear |
| Human escalation rate | trace: escalation events / total runs | trending up = harness not adapting |

### 5.6 Canary / Replay Protocol

After any harness change (new hook, new golden principle, new permission rule),
replay a fixed corpus of representative tasks and compare trajectories.

```
CANARY PROTOCOL (add to docs/CANARY_TASKS.md):

Corpus: 5 representative tasks per repo, covering:
  - Simple bug fix (unit test + code change)
  - Docs-only update
  - Protected-path change (auth/billing)
  - Multi-file refactor
  - New feature with UI proof

For each canary task, record:
  - Total retries
  - Loop-breaker triggers
  - Failure codes encountered
  - Verifier pass/fail on first attempt
  - Time to completion
  - Human escalations
  - Hook-CI divergence

Regression threshold:
  - Any metric worse by > 20% on 3+ canary tasks → block harness promotion
  - Any new LOOP_BREAKER_TRIGGERED on a previously clean task → investigate

Schedule:
  - After every harness change: run full canary suite
  - Weekly: run canary suite to detect drift
  - After every playbook version upgrade: run canary suite before adopting
```

---

## Orchestrator Layer: Scope and Limitations

This blueprint covers Claude Code's **scaffolding** and **runtime harness** layers.
The **orchestrator** layer (WORKFLOW.md, continuation hooks, daemon dispatch, multi-run
lifecycle, issue-tracker polling) is defined in the playbook (§3.4) but has
**no Claude-native equivalent** beyond what hooks and subagents provide.

| Orchestrator feature | Playbook location | Claude Code mapping |
|---|---|---|
| WORKFLOW.md policy | §3.4 | Plain repo file — not auto-loaded, not enforced by hooks |
| Continuation hooks | §3.4 | Stop hook can block premature exit; full rehydration requires external orchestrator |
| Issue tracker polling | §3.4 | Not supported natively — requires external CI/daemon (e.g., GitHub Actions, Symphony) |
| Multi-run lifecycle | §3.4 | progress.json + Initializer pattern provide session continuity; fleet dispatch is external |
| Approval workflows | §2.5 | PreToolUse hooks + permission `ask` rules provide per-tool approval; structured approve/edit/reject requires external HITL system |

**Implication:** If you need daemon-driven orchestration (background agents polling an issue tracker), you still need an external system (Symphony, GitHub Actions, custom daemon) on top of the Claude Code harness described here. This blueprint makes the interactive and CI-triggered use cases native; the daemon use case remains a playbook-level concern.

---

## What Goes Where: Quick Reference

| Playbook directive | Surface | Why here, not elsewhere |
|---|---|---|
| "Use make project-summary before deep navigation" | CLAUDE.md | Workflow norm — agent follows instructions |
| "Block reads of .env files" | settings.json deny rule | Structural enforcement — agent cannot bypass |
| "Block direct gh pr merge" | PreToolUse hook | Runtime gate — catches the action before it happens |
| "Run lint after code edits" | PostToolUse hook | Deterministic follow-up — always fires after Edit |
| "Definition of Done must be met" | Stop hook | Completion gate — blocks premature session end |
| "Scan docs for staleness" | .claude/agents/docs-maintainer.md | Specialized workflow — isolated context, restricted tools |
| "Initialize new task environment" | .claude/agents/initializer.md | First-session workflow — different prompt than coding |
| "Audit harness compliance" | .claude/skills/harness-audit/SKILL.md | On-demand — only loaded when explicitly triggered |
| "Golden principle: validate at boundaries" | docs/GOLDEN_PRINCIPLES.md | Reference doc — pointed to from CLAUDE.md |
| "Protected paths need human approval" | settings.json deny + PostToolUse | Both: deny prevents edits, hook flags for review |
| "Record mistakes as lessons" | CLAUDE.md instruction + docs/generated/lessons.md | Workflow norm — agent writes to a file; no enforcement needed |
| "Promote lessons to harness upgrades" | .claude/agents/lessons-sweep.md | Specialized workflow — groups lessons, proposes promotions, agent implements |
| "Expand prompt into full product spec" | .claude/agents/planner.md | Separate agent — scope expansion is a different skill than coding |
| "Grade work adversarially" | .claude/agents/evaluator.md | Separate agent — self-evaluation is unreliable; separation enables skepticism |
| "Negotiate sprint acceptance criteria" | .claude/handoffs/sprint-N-contract.md | File-based negotiation — auditable, readable by both agents |
| "Hand off context between sessions" | .claude/handoffs/session-N-handoff.md | File-based handoff — survives context resets, committed to git |
| "Is this harness component still needed?" | Canary tasks + simplification protocol | Re-evaluate after every model upgrade — remove dead weight |
| "Remember this correction for next time" | .claude/memory/<topic>.md | Agent memory — captured during session, consolidated by dream-sweep |
| "Consolidate accumulated memory" | .claude/agents/dream-sweep.md | Between-session consolidation — 4-phase dream cycle, dual gate trigger |

---

## Plugin Extraction: When and What

`[OPTIONAL]` — Only after the repo-native harness is stable across 2+ repos.

**Extract to a plugin (shared across repos):**
- Generic harness-audit skill
- Generic merge-gate checker framework
- Generic docs-gardener automation
- Generic session-start/stop hooks
- Generic permission templates

**Keep in the repo (never extract):**
- Architecture docs
- Golden principles (repo-specific patterns)
- Protected paths (repo-specific globs)
- Command contracts (repo-specific make targets)
- progress.json schema
- Domain-specific CLAUDE.md overrides

---

*This blueprint is an implementation guide for Claude Code. The Harness Engineering Playbook
(`.harness/PLAYBOOK_SKILL.md`) remains the source of truth for concepts, evidence, and
cross-tool patterns.*
