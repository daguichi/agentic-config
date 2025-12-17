---
description: E2E spec orchestrator with workflow modifiers (full/normal/lean/leanest)
argument-hint: "[modifier] [--skip=STEPS] <spec_path|inline_prompt>"
project-agnostic: true
---

# E2E Spec Orchestrator: /o_spec [modifier] [model_override] <spec_path>|<inline_prompt>

# TASK

IMPLEMENT PRODUCTION-READY full autonomous end-to-end features/fixes/chores leveraging spec workflow with multi-agent orchestrator management skills.

# ROLE & BEHAVIOR

- INVOKE `agent-orchestrator-manager` skill.
- PROCEED autonomously without waiting for user confirmation unless explicitly requested by the user or a decision needs to be made that you cannot make yourself.
- ALWAYS execute the ENTIRE WORKFLOW, EVEN if SPEC is an inline_prompt.
- ENSURE EACH AGENT IN EACH STEP COMMITS ITS CORRESPONDING CHANGES TRACING PROGRESS IN GIT HISTORY.
    - PREPEND each commit message with `spec(NNN):`
- NO MVP APPROACHES/PLACEHOLDERS/later TODOs are allowed. EVERY implementation MUST be production-ready.
    - PENALIZE AGENTS who DON'T OBEY this.

## CRITICAL RULES

- ONLY use `/spec <STAGE> <path>` commands - NEVER invent custom instructions
- Format: `/spawn <model> "/spec <STAGE> <path> [ultrathink]"` - NOTHING ELSE
- DEFINE CORRECT spec path for spec creation using `specs/<YYYY>/<MM>/<branch>/<NNN>-<title>.md`. PRIORITIZE using the spec path used in RECENT commits (branch).

# VARIABLES

Parse $ARGUMENTS to extract:
- MODIFIER: full (default), normal, lean, leanest
- MODEL_OVERRIDE: optional model name (opus/sonnet/haiku)
- SKIP_STEPS: optional --skip=STEP1,STEP2 flag
- SPEC: remaining arguments (file path or inline prompt)

Examples:
- `/o_spec specs/path.md` = full modifier
- `/o_spec normal specs/path.md` = normal modifier
- `/o_spec lean sonnet specs/path.md` = lean modifier with sonnet override
- `/o_spec leanest --skip=TEST,DOCUMENT specs/path.md` = leanest with skipped steps

NOTE: SPEC may be a file OR an inline-prompt.

# MODIFIERS

## FULL (default)
Quality-first: Maximum oversight and planning rigor
- Uses: opus for critical stages (CREATE, RESEARCH, PLAN, PLAN_REVIEW, REVIEW)
- Uses: sonnet for execution stages (IMPLEMENT, TEST, DOCUMENT)
- All stages with ultrathink where applicable

## NORMAL
Balanced: Removes redundant review step
- Removes: PLAN_REVIEW stage
- Uses: opus for critical stages (CREATE, RESEARCH, PLAN, REVIEW)
- Uses: sonnet for execution stages (IMPLEMENT, TEST, DOCUMENT)
- All stages with ultrathink where applicable

## LEAN
Speed-focused: Reduced model costs, skips research
WARNING: Reduced quality vs speed tradeoff
- Removes: RESEARCH stage
- Uses: sonnet for all stages
- All stages with ultrathink where applicable

## LEANEST
Maximum speed: Minimal oversight, downgrades execution models
WARNING: Significantly reduced quality for maximum speed
- Removes: RESEARCH stage
- Uses: sonnet for planning/review (CREATE, PLAN, REVIEW)
- Uses: haiku for execution (IMPLEMENT, TEST, DOCUMENT)
- No ultrathink on haiku stages

# WORKFLOW

EXECUTE SEQUENTIALLY based on MODIFIER:

## FULL WORKFLOW
1. /spawn opus "/spec CREATE {SPEC}" (SKIP if spec file exists)
2. /spawn opus "/spec RESEARCH {SPEC} ultrathink"
3. /spawn opus "/spec PLAN {SPEC} ultrathink"
4. /spawn opus "/spec PLAN_REVIEW {SPEC} ultrathink"
5. /spawn sonnet "/spec IMPLEMENT {SPEC} ultrathink"
6. /spawn opus "/spec REVIEW {SPEC} ultrathink"
7. /spawn sonnet "/spec TEST {SPEC}"
8. /spawn sonnet "/spec DOCUMENT {SPEC}"

## NORMAL WORKFLOW
1. /spawn opus "/spec CREATE {SPEC}" (SKIP if spec file exists)
2. /spawn opus "/spec RESEARCH {SPEC} ultrathink"
3. /spawn opus "/spec PLAN {SPEC} ultrathink"
4. /spawn sonnet "/spec IMPLEMENT {SPEC} ultrathink"
5. /spawn opus "/spec REVIEW {SPEC} ultrathink"
6. /spawn sonnet "/spec TEST {SPEC}"
7. /spawn sonnet "/spec DOCUMENT {SPEC}"

## LEAN WORKFLOW
1. /spawn sonnet "/spec CREATE {SPEC}" (SKIP if spec file exists)
2. /spawn sonnet "/spec PLAN {SPEC} ultrathink"
3. /spawn sonnet "/spec IMPLEMENT {SPEC} ultrathink"
4. /spawn sonnet "/spec REVIEW {SPEC} ultrathink"
5. /spawn sonnet "/spec TEST {SPEC}"
6. /spawn sonnet "/spec DOCUMENT {SPEC}"

## LEANEST WORKFLOW
1. /spawn sonnet "/spec CREATE {SPEC}" (SKIP if spec file exists)
2. /spawn sonnet "/spec PLAN {SPEC}"
3. /spawn haiku "/spec IMPLEMENT {SPEC}"
4. /spawn sonnet "/spec REVIEW {SPEC}"
5. /spawn haiku "/spec TEST {SPEC}"
6. /spawn haiku "/spec DOCUMENT {SPEC}"

## STEP SKIPPING
If --skip flag provided, exclude specified steps from workflow.
Example: `--skip=TEST,DOCUMENT` removes steps 7-8 from full workflow

CAUTION: Skipping steps reduces quality assurance and documentation coverage.

# REPORT

REPORT PROGRESSIVE CONCISE YET INSIGHTFUL updates to me, your manager, as you progress through the workflow

# CONTROLS

- IF I interrupt your worflow execution, I may use the word `resume ...` or `resume from step X ...`/`resume X ...`, which indicates you need to continue from where we left off before the interruption OR from the specific step number indicated by `X`

# NOTES

