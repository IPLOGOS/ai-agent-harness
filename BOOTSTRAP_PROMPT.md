# BOOTSTRAP PROMPT — Idempotent Harness Deployment
# Fonctionne sur : repo vierge, repo sans harness, harness partiel, harness obsolète
#
# Prérequis unique : copier le playbook dans .harness/PLAYBOOK_SKILL.md
#
# DESIGN GOAL: idempotent, minimum-diff harness deployment.
# The bootstrap preserves compliant repo-specific content and avoids unnecessary rewrites.
# However, changes remain subject to verification; "no-op on compliant repos" is a target,
# not a theorem. Agent systems still fail often enough that environment synthesis should be
# treated as uncertain work, not guaranteed-safe work.


## ─── PROMPT UNIVERSEL (copier-coller tel quel) ────────────────────────

Read the harness engineering playbook at .harness/PLAYBOOK_SKILL.md in full.
This is a skill document — internalize every [REQUIRED] and [DEFAULT] directive
before doing anything. Do NOT copy [EXAMPLE] blocks blindly.

TARGET: claude-code
(Options: claude-code | codex | generic)

  claude-code — PRIMARY output: CLAUDE.md + .claude/settings.json + .claude/agents/.
               Also read .harness/CLAUDE_CODE_BLUEPRINT.md for implementation details.
               Generate AGENTS.md as Codex compatibility mirror.
  codex      — PRIMARY output: AGENTS.md + docs/ + Makefile.
               Generate CLAUDE.md as Claude Code compatibility mirror.
  generic    — AGENTS.md-first, no tool-specific surfaces (.claude/ etc).

EXECUTION MODE: review
(Options: audit | review | apply | autonomous)

  audit      — Run PHASE 0 only. Show the gap report. Do not modify anything.
  review     — Run PHASE 0–1, wait for confirmation, then PHASE 2–3.
               This is the default and safest mode for first use.
  apply      — Run PHASE 0–3 with a single confirmation checkpoint after PHASE 1.
               Use when you trust the audit and want fewer interruptions.
  autonomous — Run PHASE 0–3 without checkpoints. Stop only if protected paths
               or policy violations are detected. Use for CI/automation or when
               re-running on a repo you've already bootstrapped.

Then execute the following phases IN ORDER, respecting the mode above.

═══════════════════════════════════════════════════════════════════════════
PHASE 0 — AUDIT: Detect current harness state
═══════════════════════════════════════════════════════════════════════════

Before generating or modifying ANYTHING, audit the current repository:

Step 0.1 — Detect harness presence:
  - Does AGENTS.md (or CLAUDE.md) exist at the repo root?
  - Does a docs/ directory exist with harness files?
  - Does a .harness/harness.meta.json version marker exist?
  - Does a .claude/ directory exist with settings.json, agents/, or skills/?
  Classify the repo as: FRESH | HAS_HARNESS | HAS_PARTIAL_HARNESS

Step 0.2 — If harness files exist, extract the version and target:
  Look for these fields in .harness/harness.meta.json:
    "playbook_version": "X.Y.Z"
    "target": "claude-code | codex | generic"
  If no version is found, classify as: VERSION_UNKNOWN (treat as oldest).
  If target differs from the current TARGET parameter, note the mismatch.

Step 0.3 — Compare the installed version against the playbook version.
  The playbook version is declared at the top of .harness/PLAYBOOK_SKILL.md:
    "Playbook version: X.Y.Z"

Step 0.4 — Run the compliance checklist.
  For EVERY file and target listed in Appendix B (File Manifest) and
  Appendix A (Quick-Start Checklist) of the playbook, check:
    EXISTS?     — does the file/target exist?
    COMPLIANT?  — does it contain [REQUIRED] elements from the playbook?
    CURRENT?    — does it have a valid staleness header (< 90 days)?

Step 0.5 — Produce a GAP REPORT (do NOT fix anything yet).
  Output the report as a structured list:

  ## Harness Audit Report
  **Repo state:** FRESH | HAS_HARNESS | HAS_PARTIAL_HARNESS
  **Installed version:** X.Y.Z | VERSION_UNKNOWN | NONE
  **Playbook version:** X.Y.Z
  **Upgrade needed:** YES | NO

  ### Files
  | File | Status | Issue |
  |------|--------|-------|
  | AGENTS.md | MISSING / PRESENT / OUTDATED | details |
  | docs/ARCHITECTURE.md | ... | ... |
  | ... | ... | ... |

  ### Command Contracts
  | Target | Status | Issue |
  |--------|--------|-------|
  | make project-summary | MISSING / PRESENT / NON-COMPLIANT | details |
  | make find-refs | ... | ... |
  | ... | ... | ... |

  ### Playbook Directives
  | Directive | Status | Issue |
  |-----------|--------|-------|
  | Code Navigation Rule in root map | PRESENT / MISSING | ... |
  | Loop Breaker policy in root map | PRESENT / MISSING | ... |
  | Tool output offloading rule | PRESENT / MISSING | ... |
  | Initializer instructions | PRESENT / MISSING | ... |
  | Tool preference order | PRESENT / MISSING | ... |
  | Staleness headers on all docs | X/Y compliant | ... |
  | progress.json status constraints | COMPLIANT / NON-COMPLIANT / N/A | ... |
  | Continuation hooks in WORKFLOW.md | PRESENT / MISSING / N/A | ... |
  | ... | ... | ... |

  ### Claude Code Surfaces (if TARGET = claude-code)
  | Surface | Status | Issue |
  |---------|--------|-------|
  | .claude/settings.json | MISSING / PRESENT / OUTDATED | ... |
  | .claude/settings.json hooks | X/4 configured | ... |
  | .claude/agents/initializer.md | MISSING / PRESENT | ... |
  | .claude/agents/verifier.md | MISSING / PRESENT | ... |
  | .claude/agents/docs-maintainer.md | MISSING / PRESENT | ... |
  | .claude/agents/merge-checker.md | MISSING / PRESENT | ... |
  | .claude/agents/security-reviewer.md | MISSING / PRESENT | ... |
  | scripts/hooks/ (4 hook scripts) | X/4 present | ... |
  | Domain CLAUDE.md files | X present for Y domains | ... |
  | ... | ... | ... |

  Present this report.
  - In "audit" mode: STOP here. Do not proceed.
  - In "review" mode: WAIT for confirmation before proceeding.
  - In "apply" mode: proceed to PHASE 1 automatically.
  - In "autonomous" mode: proceed to PHASE 1 automatically.
  If the report shows "Upgrade needed: NO" and all files are COMPLIANT,
  say "Harness is up to date — no changes needed." and stop in ALL modes.

═══════════════════════════════════════════════════════════════════════════
PHASE 1 — PLAN: Determine what to create, update, or leave alone
═══════════════════════════════════════════════════════════════════════════

Based on the gap report, classify each item:

  CREATE   — file/target does not exist → generate from scratch
  UPDATE   — file/target exists but is non-compliant or outdated → patch
  SKIP     — file/target exists and is compliant → do not touch

For UPDATE items:
  - PRESERVE all existing content that is repo-specific (architecture docs
    that describe this codebase's actual structure, golden principles that
    reference real patterns, custom make targets, etc.)
  - ADD only the missing [REQUIRED] elements from the playbook
  - Do NOT overwrite repo-specific customizations with generic templates
  - If a doc has valid content but is missing a staleness header, add ONLY
    the header — do not rewrite the content

Present the action plan.
  - In "review" mode: WAIT for confirmation before executing.
  - In "apply" mode: WAIT for confirmation before executing.
  - In "autonomous" mode: proceed to PHASE 2 automatically
    UNLESS the plan includes UPDATE items on protected paths
    (src/auth/**, src/billing/**, infra/**, migrations/**),
    in which case WAIT for confirmation regardless of mode.

═══════════════════════════════════════════════════════════════════════════
PHASE 2 — EXECUTE: Apply changes (target-aware)
═══════════════════════════════════════════════════════════════════════════

For each CREATE item:
  Generate the file following the playbook's specifications.
  Adapt to THIS repo's actual stack, structure, and conventions.
  Do not hallucinate architecture — only document what actually exists.
  If something is uncertain, mark it with <!-- TODO: verify -->.

For each UPDATE item:
  Make the minimum change needed to reach compliance.
  Show a diff-style summary of what changed and why.

──── IF TARGET = claude-code ────────────────────────────────────────────

  Read .harness/CLAUDE_CODE_BLUEPRINT.md for the full implementation spec.

  BOOTSTRAP PHASING (do not run all passes in one shot):
    Pass 1 (ALWAYS): Minimum viable runtime — CLAUDE.md, settings.json,
      initializer, portable helpers, progress. Stop here for new repos.
    Pass 2 (AFTER Pass 1 proves stable): Repo docs + command contracts.
    Pass 3 (AFTER Pass 2): Hook enforcement — the harness becomes mechanical.
    Pass 4 (AFTER Pass 3): Subagents + skills + slash commands.
    Pass 5 (AFTER Pass 4): Canary tasks + observability verification.
    Pass 6 (OPTIONAL): Codex compatibility mirror.

  For FRESH repos: execute ONLY Pass 1. Verify the harness works
  (make test, make lint, agent can follow CLAUDE.md). Then run the
  bootstrap again with EXECUTION MODE: apply for subsequent passes.

  For EXISTING harness repos: the gap report (PHASE 0) determines
  which passes have gaps. Execute only the passes with gaps.

  Primary root map: CLAUDE.md
    [REQUIRED] Must include ALL of these (add any that are missing):
      - Commands, Architecture/Conventions/Merge Gates/Security pointers
      - Code Navigation Rule, Loop Breaker, Session Start instructions
      - Tool output offloading rule, Tool hierarchy, External knowledge rule
    [DEFAULT] Should stay under ~100 lines; up to ~200 if highly structured.
    [REQUIRED] Must be a pointer map, not an encyclopedia.

  Claude-native surfaces (generate per the Blueprint's Bucket classification):
    .claude/settings.json     — permissions (deny/ask rules) + hooks
    .claude/agents/           — initializer.md, verifier.md, docs-maintainer.md,
                                merge-checker.md, security-reviewer.md,
                                lessons-sweep.md, dream-sweep.md,
                                planner.md, evaluator.md
    .claude/commands/         — audit-harness.md, init-task.md, promote-lessons.md
    .claude/handoffs/         — inter-agent communication directory (initially empty)
    .claude/memory/           — agent memory directory with MEMORY.md index + dream-state.json
    scripts/lib/              — portable-hash.sh, portable-uuid.sh
    src/<domain>/CLAUDE.md    — per-subsystem overrides (< 50 lines each)
    scripts/hooks/            — session-start.sh, pre-tool-use-guard.sh,
                                post-edit-validate.sh, stop-gate.sh

  Codex compatibility mirror:
    Generate AGENTS.md from CLAUDE.md content (same commands, same pointers).

──── IF TARGET = codex ──────────────────────────────────────────────────

  Primary root map: AGENTS.md
    [REQUIRED] Same content requirements as above.
    [DEFAULT] Should stay under ~100 lines.

  Claude Code compatibility mirror:
    Generate CLAUDE.md from AGENTS.md content.

  No .claude/ directory is generated (not applicable to Codex).

──── IF TARGET = generic ────────────────────────────────────────────────

  Primary root map: AGENTS.md
  Also generate: CLAUDE.md as mirror.
  No tool-specific surfaces.

──── COMMON TO ALL TARGETS ──────────────────────────────────────────────

For command contracts (Makefile targets / scripts):
  If a target already exists, verify it matches the command contract spec:
    - Has documented purpose, inputs, output, exit codes, on-failure
    - Exit codes are correct
    - Failure behavior matches playbook (e.g., Loop Breaker retry limits)
  If non-compliant: update the target AND its documentation comment.
  If compliant: SKIP.

For docs/ (all targets):
  Generate all docs from Appendix B (File Manifest) of the playbook.
  Every doc must have a staleness header.
  Include docs/OBSERVABILITY.md (failure codes, correlation schema, evidence format)
  and docs/CANARY_TASKS.md (canary task corpus — 5 representative tasks).
  Include docs/generated/lessons.md (empty template with format instructions).

For FRESH repos that need the Initializer pattern (§1.5):
  Also generate: init.sh, progress.json with feature list (all "pending"),
  and make the initial git commit as a clean baseline.

═══════════════════════════════════════════════════════════════════════════
PHASE 3 — VERSION STAMP & VERIFY
═══════════════════════════════════════════════════════════════════════════

Step 3.1 — Write the version marker:
  Create or update .harness/harness.meta.json:
  {
    "playbook_version": "<version from playbook>",
    "target": "<claude-code | codex | generic>",
    "last_audit": "<today's date>",
    "last_audit_commit": "<current short SHA>",
    "state": "COMPLIANT",
    "items_created": [ <list> ],
    "items_updated": [ <list> ],
    "items_skipped": [ <list> ]
  }

Step 3.2 — Compatibility mirror:
  If TARGET = claude-code: ensure AGENTS.md mirrors CLAUDE.md
  If TARGET = codex: ensure CLAUDE.md mirrors AGENTS.md
  If TARGET = generic: ensure both exist and match

Step 3.3 — Verification:
  Run: make lint && make test
  If either fails ON HARNESS-INTRODUCED CHANGES:
    Fix the issue. The harness must not break existing code.
  Run: make check-docs (if the target exists now)
    Verify all docs have staleness headers.
  Run: make project-summary (if the target exists now)
    Verify output is < 200 lines and contains no file bodies.

Step 3.4 — Commit:
  If FRESH or HAS_PARTIAL_HARNESS:
    git add -A && git commit -m "chore: scaffold harness engineering framework (playbook vX.Y.Z)"

  If upgrading from a previous version:
    git add -A && git commit -m "chore: upgrade harness to playbook vX.Y.Z

    Created:
    - [list CREATE items]
    Updated:
    - [list UPDATE items with brief description of change]
    Skipped (already compliant):
    - [list SKIP items]"

  If already compliant (no changes):
    No commit needed.

═══════════════════════════════════════════════════════════════════════════
RULES (apply throughout all phases)
═══════════════════════════════════════════════════════════════════════════

- NEVER delete existing repo-specific content. The harness adapts to the repo,
  not the other way around.
- NEVER overwrite a compliant file with a generic template.
- Analyze the actual codebase BEFORE generating architecture or golden principles.
- Golden principles must reference REAL patterns found in this codebase.
- If the repo has no tests, create docs/TESTING.md with a plan, not fake coverage.
- Every doc must have a staleness header:
  <!-- Last verified: YYYY-MM-DD | Commit: SHORT_SHA | Status: CURRENT -->
- Items marked [OPTIONAL] in the playbook should be SKIPPED unless specifically requested.
- If you encounter a harness file that has custom additions beyond the playbook
  (team-specific gates, extra golden principles, custom automations):
  PRESERVE them. They are house policy. Only add what's missing from [REQUIRED].
- IDEMPOTENT GOAL: running this on a compliant repo should change nothing.
  However, this is a design target, not a theorem. Agent-driven repo surgery
  is uncertain work. Always verify output with make lint && make test.
- DETERMINISTIC HELPERS: For version comparison, date math, staleness header
  validation, and SHA matching, prefer deterministic shell commands over
  reasoning about values in prose. Examples:
    Version comparison: use sort -V or semver scripts, not mental parsing.
    Staleness date math: use date +%s arithmetic, not prose date reasoning.
    File existence/content checks: use test/jq/grep, not recall.
  The model should execute checks, not compute them in-context.


## ─── RACCOURCIS ────────────────────────────────────────────────────────
## Pour les cas courants, ces variantes courtes fonctionnent :

### Audit seul (ne rien modifier) :

Read .harness/PLAYBOOK_SKILL.md.
EXECUTION MODE: audit
Run PHASE 0 only. Show the gap report. Do not modify anything.

### Mise à jour après upgrade du playbook (avec review) :

Read .harness/PLAYBOOK_SKILL.md. The playbook has been updated.
EXECUTION MODE: review
Run all phases. Upgrade the harness to match the new version.
Preserve all repo-specific content. Only add missing [REQUIRED] elements.

### Mise à jour rapide (confiance haute, moins d'interruptions) :

Read .harness/PLAYBOOK_SKILL.md. The playbook has been updated.
EXECUTION MODE: apply
Run all phases. Show plan once, then execute after confirmation.

### Re-run en CI / automation (aucun checkpoint sauf protected paths) :

Read .harness/PLAYBOOK_SKILL.md.
TARGET: claude-code
EXECUTION MODE: autonomous
Run all phases without checkpoints unless protected paths are touched.

### Claude Code — first deployment :

Read .harness/PLAYBOOK_SKILL.md.
Also read .harness/CLAUDE_CODE_BLUEPRINT.md for implementation details.
TARGET: claude-code
EXECUTION MODE: review
Run all phases. Generate CLAUDE.md as primary, .claude/ surfaces, AGENTS.md as mirror.

### Codex — first deployment :

Read .harness/PLAYBOOK_SKILL.md.
TARGET: codex
EXECUTION MODE: review
Run all phases. Generate AGENTS.md as primary, CLAUDE.md as mirror.

### Nouveau projet :

Read .harness/PLAYBOOK_SKILL.md.
EXECUTION MODE: review
This is a NEW repository. Run all phases including the Initializer (§1.5):
create init.sh, progress.json, and the initial git commit.
The project is: [DÉCRIRE] | Stack: [STACK] | Domains: [DOMAINES]

### Vérification post-bootstrap (nouvelle session, NE PAS mentionner le harness) :

There is a small bug: the login page shows a flash of unstyled content on first load.
Fix it.

# L'agent devrait suivre le harness automatiquement via AGENTS.md auto-chargé :
#   ✓ make project-summary pour s'orienter
#   ✓ make find-refs plutôt que cat **/*.ts
#   ✓ Loop Breaker si bloqué
#   ✓ Proof artifacts
#   ✓ Merge gates
# Si tout ça se produit sans mention du harness → déploiement réussi.


## ─── SCÉNARIOS ─────────────────────────────────────────────────────────
##
##  Scénario 1 : Repo vierge
##    PHASE 0 → "FRESH, no harness"
##    PHASE 1 → tout en CREATE
##    PHASE 2 → génère tout, adapté au projet décrit
##    PHASE 3 → version stamp + initial commit
##
##  Scénario 2 : Repo existant, pas de harness
##    PHASE 0 → "HAS_PARTIAL_HARNESS" (Makefile, tests existent peut-être)
##    PHASE 1 → CREATE les fichiers harness, SKIP les targets existants conformes
##    PHASE 2 → génère ce qui manque, adapté au repo
##    PHASE 3 → version stamp + commit
##
##  Scénario 3 : Harness v3.3 installé, playbook mis à jour en v3.4.0
##    PHASE 0 → "HAS_HARNESS, version 3.1, upgrade needed"
##    PHASE 0 → gap report liste les éléments manquants de v3.4.0
##              (ex: tool offloading rule, continuation hooks, staleness headers...)
##    PHASE 1 → UPDATE pour les manques, SKIP pour le reste
##    PHASE 2 → ajoute SEULEMENT ce qui manque, préserve le repo-specific
##    PHASE 3 → version stamp → "chore: upgrade harness to playbook v3.4.0"
##
##  Scénario 4 : Harness déjà conforme et à jour
##    PHASE 0 → "COMPLIANT, no changes needed"
##    → STOP. Rien à faire.
##
##  Scénario 5 : Harness partiel (ex: AGENTS.md existe mais pas MERGE_GATES.md)
##    PHASE 0 → "HAS_PARTIAL_HARNESS, VERSION_UNKNOWN"
##    PHASE 0 → gap report montre quels fichiers manquent / sont non-conformes
##    PHASE 1 → CREATE les manquants, UPDATE les non-conformes, SKIP les bons
##    PHASE 2 → comble les trous sans toucher ce qui marche
##    PHASE 3 → version stamp + commit listant les changements
