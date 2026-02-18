<purpose>
Execute all plans in a phase using a Claude Code agent team. Executor teammates coordinate
via a shared task list — each claims work autonomously, respects wave dependencies, and
communicates progress directly to the lead. Verifier runs as a subagent after all plans complete.

Use when: phase has multiple plans across waves and you want parallel execution with
inter-agent coordination. For single-plan phases, /gsd:execute-phase is simpler.
</purpose>

<when_to_use_teams>
Teams add real value here because:
- Executors self-claim tasks as wave dependencies resolve — no lead micromanagement
- Executors can message each other when one plan's output affects another
- User can interact with individual executors directly during execution
- Lead stays lean — only handles checkpoints, not plan routing

Cost: more tokens than subagents. Worth it for 3+ plans across multiple waves.
</when_to_use_teams>

<required_reading>
Read STATE.md before any operation to load project context.
</required_reading>

<process>

<step name="initialize" priority="first">
Load all context in one call:

```bash
INIT=$(node ~/.claude/get-shit-done/bin/gsd-tools.cjs init execute-phase "${PHASE_ARG}")
LANGUAGE_INSTRUCTION=$(echo "$INIT" | jq -r '.language_instruction // empty')
```

Parse JSON for: `executor_model`, `verifier_model`, `commit_docs`, `parallelization`,
`branching_strategy`, `branch_name`, `phase_found`, `phase_dir`, `phase_number`,
`phase_name`, `phase_slug`, `plans`, `incomplete_plans`, `plan_count`, `incomplete_count`,
`state_exists`, `roadmap_exists`, `language_instruction`.

**If `phase_found` is false:** Error — phase directory not found.
**If `plan_count` is 0:** Error — no plans found in phase.

**Check teams enabled:**

```bash
TEAMS_CFG=$(CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1 node -e "console.log(process.env.CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS || '0')" 2>/dev/null || echo "0")
```

If agent teams appear unavailable (TeamCreate tool not present), display:
```
⚠ Agent teams require CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1 in your environment.

Add to ~/.claude/settings.json:
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}

Then restart Claude Code and re-run this command.
```
And exit.
</step>

<step name="handle_branching">
Check `branching_strategy` from init:

**"none":** Skip, continue on current branch.

**"phase" or "milestone":** Use pre-computed `branch_name` from init:
```bash
git checkout -b "$BRANCH_NAME" 2>/dev/null || git checkout "$BRANCH_NAME"
```
</step>

<step name="discover_and_group_plans">
Load plan inventory with wave grouping:

```bash
PLAN_INDEX=$(node ~/.claude/get-shit-done/bin/gsd-tools.cjs phase-plan-index "${PHASE_NUMBER}")
```

Parse JSON for: `phase`, `plans[]` (each with `id`, `wave`, `autonomous`, `objective`,
`files_modified`, `task_count`, `has_summary`), `waves` (map of wave → plan IDs),
`incomplete`, `has_checkpoints`.

**Filtering:** Skip plans where `has_summary: true`.
If all filtered: "No incomplete plans found" → exit.

Report:
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► TEAM EXECUTION: Phase {N} — {phase_name}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

{plan_count} plans | {wave_count} waves | Team mode

| Wave | Plans | What it builds |
|------|-------|----------------|
| 1 | {plan IDs} | {objectives, 3-8 words} |
| 2 | {plan IDs} | ... |

◆ Creating team and task list...
```
</step>

<step name="create_team_and_tasks">
**Create the team:**

```
TeamCreate(
  team_name="gsd-phase-{PHASE_NUMBER}",
  description="Execute Phase {PHASE_NUMBER}: {phase_name}",
  agent_type="team-lead"
)
```

**Create one task per plan, respecting wave dependencies:**

For each plan in wave order:
- Determine which task IDs must complete before this one (all tasks in previous waves)
- Create the task:

```
TaskCreate(
  subject="Execute plan {plan_id}: {plan_name}",
  description="Execute plan at {phase_dir}/{plan_file}. Follow execute-plan.md. Commit each task atomically. Create SUMMARY.md. Update STATE.md and ROADMAP.md.",
  activeForm="Executing plan {plan_id}"
)
```

Store the returned task ID for dependency tracking.

For wave 2+ plans, after creating, set blockedBy:
```
TaskUpdate(taskId="{task_id}", addBlockedBy=["{wave_1_task_id}", ...])
```

**Create the verification task** (blocked by all plan tasks):
```
TaskCreate(
  subject="Verify Phase {PHASE_NUMBER} goal achievement",
  description="Verify phase achieved its GOAL, not just completed tasks. Read all SUMMARYs and check codebase. Create VERIFICATION.md.",
  activeForm="Verifying phase goal"
)
TaskUpdate(taskId="{verify_task_id}", addBlockedBy=[{all_plan_task_ids}])
```

Display task list after creation:
```
◆ Task list created:

| Task | Plan | Wave | Status |
|------|------|------|--------|
| {id} | {plan_id} | 1 | pending |
| {id} | {plan_id} | 1 | pending |
| {id} | {plan_id} | 2 | blocked |
| {verify_id} | verify | — | blocked |

◆ Spawning executor teammates...
```
</step>

<step name="spawn_executors">
Spawn one executor teammate per plan in wave 1 (cap at 3 executors max).
If wave 1 has more than 3 plans, spawn 3 — they will self-claim remaining tasks.

For each executor, use this prompt template:

```
Task(
  subagent_type="gsd-executor",
  model="{executor_model}",
  team_name="gsd-phase-{PHASE_NUMBER}",
  name="executor-{N}",
  description="Execute Phase {PHASE_NUMBER} plans",
  prompt="
<role>
You are executor-{N}, a GSD executor teammate in team gsd-phase-{PHASE_NUMBER}.
You are executing Phase {PHASE_NUMBER}: {phase_name}.
</role>

<execution_context>
@~/.claude/get-shit-done/workflows/execute-plan.md
@~/.claude/get-shit-done/templates/summary.md
@~/.claude/get-shit-done/references/checkpoints.md
@~/.claude/get-shit-done/references/tdd.md
</execution_context>

<your_first_task>
Start with this plan:
- Plan ID: {plan_id}
- Plan file: {phase_dir}/{plan_file}
- Task list task ID: {task_id}

Read the plan file, execute it following execute-plan.md, then mark your task complete.
</your_first_task>

<team_workflow>
After completing your first plan:

1. Call TaskUpdate to mark your task completed:
   TaskUpdate(taskId='{task_id}', status='completed')

2. Call TaskList to find next available task (status: pending, no owner, not blocked).
   Prefer tasks in lowest ID order.

3. If an unblocked execution task is available: claim it with TaskUpdate(owner='executor-{N}'),
   then execute it.

4. Repeat until no more execution tasks remain.

5. When no more tasks: SendMessage to the team lead:
   'EXECUTION COMPLETE: executor-{N} finished. Plans completed: {list}. All tasks marked done.'

Do NOT claim or execute the verification task — that belongs to the verifier.
</team_workflow>

<checkpoint_protocol>
If a plan has a checkpoint task (autonomous: false):
- Execute up to the checkpoint
- SendMessage to team lead: 'CHECKPOINT: {checkpoint details} in plan {plan_id}. Awaiting your response.'
- Wait for the lead's reply before continuing
</checkpoint_protocol>

<files_to_read_at_start>
- Plan: {phase_dir}/{plan_file}
- State: .planning/STATE.md
- Config: .planning/config.json
</files_to_read_at_start>
" + (language_instruction ? "\n\n" + language_instruction : "")
)
```
</step>

<step name="monitor_execution">
Wait for messages from executor teammates. Messages arrive automatically — no polling needed.

**Message routing:**

| Message type | Action |
|--------------|--------|
| `EXECUTION COMPLETE: executor-{N}...` | Mark that executor done, track which plans it finished |
| `CHECKPOINT: ...` | Follow checkpoint handling (see below) |
| Idle notification from all executors | Check if all plan tasks are marked completed in TaskList |

**Track completion:**

Maintain a count of plan tasks completed. Check TaskList periodically to confirm task statuses.

When all plan tasks are `completed` in TaskList:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► ALL PLANS COMPLETE — Starting verification
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Proceed to `run_verifier`.
</step>

<step name="handle_checkpoints">
When an executor sends a CHECKPOINT message:

1. Parse: plan ID, checkpoint type (human-verify | decision | human-action), checkpoint details

2. Present to user:
```
╔══════════════════════════════════════════════════════════════╗
║  TEAM CHECKPOINT — executor-{N}                              ║
╚══════════════════════════════════════════════════════════════╝

**Plan:** {plan_id}
**Type:** {checkpoint_type}

{checkpoint_details}

──────────────────────────────────────────────────────────────
→ Type "approved" / decision / describe issue
──────────────────────────────────────────────────────────────
```

3. After user responds:
```
SendMessage(
  type="message",
  recipient="executor-{N}",
  content="Checkpoint response: {user_response}. Continue execution.",
  summary="Checkpoint approved"
)
```

4. Resume monitoring.
</step>

<step name="run_verifier">
After all plan tasks complete, spawn verifier as a subagent (not a teammate):

```bash
PHASE_REQ_IDS=$(node ~/.claude/get-shit-done/bin/gsd-tools.cjs roadmap get-phase "${PHASE_NUMBER}" | jq -r '.section' | grep -i "Requirements:" | sed 's/.*Requirements:\*\*\s*//' | sed 's/[\[\]]//g')
```

```
Task(
  prompt="Verify phase {phase_number} goal achievement.
Phase directory: {phase_dir}
Phase goal: {goal from ROADMAP.md}
Phase requirement IDs: {phase_req_ids}
Check must_haves against actual codebase.
Cross-reference requirement IDs from PLAN frontmatter against REQUIREMENTS.md.
Create VERIFICATION.md." + (language_instruction ? "\n\n" + language_instruction : ""),
  subagent_type="gsd-verifier",
  model="{verifier_model}"
)
```

Read status:
```bash
grep "^status:" "$PHASE_DIR"/*-VERIFICATION.md | cut -d: -f2 | tr -d ' '
```

| Status | Action |
|--------|--------|
| `passed` | → clean_up_team, then update_roadmap |
| `human_needed` | Present items for human testing, get approval or feedback |
| `gaps_found` | Present gap summary, offer `/gsd:plan-phase {phase} --gaps` |
</step>

<step name="clean_up_team">
Before updating roadmap, clean up the team:

Shut down all active executor teammates:
```
SendMessage(type="shutdown_request", recipient="executor-1", content="Phase execution complete. Thank you.")
SendMessage(type="shutdown_request", recipient="executor-2", content="Phase execution complete. Thank you.")
...
```

Wait for shutdown confirmations, then:
```
TeamDelete()
```
</step>

<step name="aggregate_results">
Spot-check executor completions:

For each SUMMARY.md:
- Verify first 2 files from `key-files.created` exist on disk
- Check `git log --oneline --all --grep="{phase}-{plan}"` returns ≥1 commit
- Check for `## Self-Check: FAILED` marker

Report:
```
## Phase {X}: {Name} — Team Execution Complete

**Executors:** {N} teammates | **Plans:** {M}/{total} complete

| Plan | Executor | Status | What was built |
|------|----------|--------|----------------|
| {N}-01 | executor-1 | ✓ | {from SUMMARY.md} |
| {N}-02 | executor-2 | ✓ | {from SUMMARY.md} |

### Issues Encountered
[Aggregate from SUMMARYs, or "None"]
```
</step>

<step name="update_roadmap">
Mark phase complete:

```bash
COMPLETION=$(node ~/.claude/get-shit-done/bin/gsd-tools.cjs phase complete "${PHASE_NUMBER}")
```

Extract: `next_phase`, `next_phase_name`, `is_last_phase`.

```bash
node ~/.claude/get-shit-done/bin/gsd-tools.cjs commit "docs(phase-{X}): complete phase execution via team" --files .planning/ROADMAP.md .planning/STATE.md .planning/REQUIREMENTS.md .planning/phases/{phase_dir}/*-VERIFICATION.md
```
</step>

<step name="offer_next">
**Auto-advance:** Check `--auto` flag and `workflow.auto_advance` config.

If auto-advance AND verification passed:
- Read and follow `~/.claude/get-shit-done/workflows/transition.md`

Otherwise:
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► PHASE {X} COMPLETE ✓ (Team execution)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Phase {X}: {Name}** — All plans executed, goal verified.

───────────────────────────────────────────────────────────────

## ▶ Next Up

**Phase {X+1}: {Next Phase Name}** — {goal}

/gsd:discuss-phase {X+1} — gather context first
/gsd:plan-phase {X+1} — plan directly

<sub>/clear first → fresh context window</sub>

───────────────────────────────────────────────────────────────
```
</step>

</process>

<team_vs_subagent>
| Aspect | /gsd:execute-phase | /gsd:team-execute-phase |
|--------|-------------------|------------------------|
| Executor lifecycle | Spawned per plan, dies after | Persistent teammate, works through tasks |
| Coordination | Lead manages wave sequencing | Task list + self-claiming |
| Communication | Results returned to lead | Direct messaging between lead and executors |
| Checkpoints | Lead spawns fresh continuation | Lead messages existing executor teammate |
| Token cost | Lower | Higher (separate context per teammate) |
| Best for | 1-3 plans, simple phases | 4+ plans, multi-wave, complex coordination |
</team_vs_subagent>

<failure_handling>
- **Executor goes idle without completing tasks:** Check TaskList — if tasks remain unclaimed, send message to idle executor or spawn a replacement
- **TeamCreate fails (teams not enabled):** Display enable instructions and exit
- **Teammate unresponsive:** Use TaskUpdate to manually mark their task and spawn replacement
- **classifyHandoffIfNeeded bug:** Same as execute-phase — spot-check SUMMARY.md and commits, treat as success if checks pass
- **TeamDelete fails (active teammates):** Shut them down first via SendMessage shutdown_request
</failure_handling>

<resumption>
If interrupted: Re-run `/gsd:team-execute-phase {phase}` → discover_plans skips completed SUMMARYs →
creates new team → resumes from first incomplete plan.

Note: previous team context is not recoverable (teams limitation). New team is created fresh.
</resumption>

<success_criteria>
- [ ] Phase context loaded and plans discovered
- [ ] TeamCreate succeeded
- [ ] One task per plan created with correct wave dependencies
- [ ] Verification task created, blocked by all plan tasks
- [ ] Executor teammates spawned (1 per wave-1 plan, max 3)
- [ ] Executors self-claim tasks and complete them
- [ ] Checkpoints handled via direct messaging
- [ ] All plan tasks marked completed in task list
- [ ] Verifier spawned after all plans done
- [ ] VERIFICATION.md created
- [ ] TeamDelete called after verification
- [ ] Roadmap updated, STATE.md advanced
- [ ] User knows next step
</success_criteria>
