# Spec: Team-Based Parallel Research

## Goal

Replace the fire-and-forget research subagents in `new-project.md` and `new-milestone.md`
with a team of **persistent researcher teammates** that:

1. Each investigate their dimension (Stack / Features / Architecture / Pitfalls)
2. **Message each other** when they find cross-dimension findings
3. Produce richer, more consistent output than isolated subagents
4. Let the user interact with individual researchers mid-research ("dig deeper into architecture")

Controlled by a new config flag `workflow.team_research` (default: `false`).
When disabled, the existing 4 × `Task()` behavior is unchanged.

---

## Background

### Current behavior (unchanged when `team_research: false`)

In both `new-project.md` (step 6) and `new-milestone.md` (step 8):

```
Spawn 4 parallel Task() subagents → each writes one research file → synthesizer Task() reads all → writes SUMMARY.md
```

Each researcher is a fire-and-forget subagent. They cannot communicate with each other.
If the Architecture researcher finds the proposed stack won't support a key feature,
it can only note this in its own file — the Stack researcher never knows.

### New behavior (when `team_research: true`)

```
TeamCreate → 4 researcher teammates → each writes file + can SendMessage to others
           → synthesizer subagent reads all files → writes SUMMARY.md → TeamDelete
```

**Why teammates > subagents for research:**

| | Subagents (current) | Teammates (new) |
|-|--------------------|--------------------|
| Communication | None — each works in isolation | Direct messaging between researchers |
| Cross-validation | Only at synthesis stage | Real-time, during research |
| User interaction | Cannot interact mid-research | User can message any researcher directly |
| Consistency | Each researcher makes independent assumptions | Shared findings reduce contradictions |
| Token cost | Lower | Higher (4 separate Claude contexts) |

---

## Files to Edit

### 1. `get-shit-done/templates/config.json`

**Add `team_research` to the defaults object.** Place it near the other `workflow.*` booleans.

Find the block containing `"research": true` and add the new field after it:

```json
"research": true,
"team_research": false,
```

Full resulting relevant section (for reference):

```json
{
  "mode": "interactive",
  "depth": "standard",
  "language": "auto",
  "research": true,
  "team_research": false,
  "plan_check": true,
  "verifier": true,
  ...
}
```

---

### 2. `get-shit-done/bin/gsd-tools.cjs`

Three changes in this file.

#### 2a. `loadConfig()` — add default

Find the `defaults` object inside `loadConfig()`. It already has:

```javascript
research: true,
```

Add the new default immediately after:

```javascript
research: true,
team_research: false,
```

And in the `return` block of `loadConfig()`, add:

```javascript
team_research: get('team_research') ?? defaults.team_research,
```

Place it immediately after the existing `research:` line in the return.

#### 2b. `cmdInitNewProject()` — expose the flag

In `cmdInitNewProject`, the `result` object currently does NOT include `research_enabled`.
Add `team_research` to the result, right after the language lines at the bottom:

```javascript
result.team_research = config.team_research;
result.language = config.language;
result.language_instruction = getLanguageInstruction(config.language);
```

#### 2c. `cmdInitNewMilestone()` — expose the flag

In `cmdInitNewMilestone`, the result already has `research_enabled: config.research`.
Add `team_research` next to it:

```javascript
research_enabled: config.research,
team_research: config.team_research,
```

---

### 3. `get-shit-done/workflows/settings.md`

**Add `team_research` toggle to the settings questions and config update.**

#### 3a. In the "Parse current values" step

The file lists which config keys to parse. Add:

```
- `workflow.team_research` — use agent teams for parallel research (default: `false`)
```

#### 3b. In `present_settings` — add a new question

The AskUserQuestion block currently has 7 questions. Add an 8th after the language question:

```
{
  question: "Use agent teams for parallel research? (experimental — requires CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1)",
  header: "Team Research",
  multiSelect: false,
  options: [
    {
      label: "Off (Recommended)",
      description: "Standard parallel subagents — faster, lower token cost"
    },
    {
      label: "On",
      description: "Researcher teammates communicate with each other — richer findings, higher cost"
    }
  ]
}
```

Pre-select based on current value: "Off" if `team_research` is false, "On" if true.

#### 3c. In the `update_config` step

Map the answer to a config value:

```
"Off (Recommended)" or "Off" → false
"On" → true
```

Write to config:

```bash
node ~/.claude/get-shit-done/bin/gsd-tools.cjs config-set workflow.team_research {true|false}
```

#### 3d. In the `save_as_defaults` JSON

Add the field to the defaults JSON written to `~/.gsd/defaults.json`:

```json
"team_research": true|false,
```

#### 3e. In the confirmation table

Add a row:

```
| Team Research        | {on/off}            |
```

#### 3f. Update success_criteria

Change "7 settings" to "8 settings" in the criteria checklist.

---

### 4. `get-shit-done/workflows/new-project.md`

**Replace the research spawn block with a conditional that checks `team_research`.**

**Where:** Step 6 "Research", specifically the section that currently reads:

```
Spawn 4 parallel gsd-project-researcher agents with rich context:
```

#### What to change

The entire block from `Spawn 4 parallel gsd-project-researcher agents...` through the
synthesizer `Task()` call needs to be wrapped in a conditional:

```
**If `team_research` is false (default):** [existing 4 × Task() + synthesizer Task() — unchanged]

**If `team_research` is true:** [new team-based flow below]
```

#### New team-based flow (insert when `team_research` is true)

```
Display:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► TEAM RESEARCH
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

◆ Creating research team...
  → Stack researcher
  → Features researcher
  → Architecture researcher
  → Pitfalls researcher
```

**Step A: Create team and tasks**

```
TeamCreate(
  team_name="gsd-research-{project_slug}",
  description="Research for {project_name}",
  agent_type="research-lead"
)

TaskCreate(subject="Stack research",        description="Research the standard 2025 stack for {domain}. Write to .planning/research/STACK.md.", activeForm="Researching stack")
TaskCreate(subject="Features research",     description="Research features for {domain} products. Write to .planning/research/FEATURES.md.", activeForm="Researching features")
TaskCreate(subject="Architecture research", description="Research architecture patterns for {domain}. Write to .planning/research/ARCHITECTURE.md.", activeForm="Researching architecture")
TaskCreate(subject="Pitfalls research",     description="Research common mistakes building {domain} apps. Write to .planning/research/PITFALLS.md.", activeForm="Researching pitfalls")
```

**Step B: Spawn 4 researcher teammates in parallel**

Spawn all 4 in a single message (parallel). Each uses this prompt structure:

```
Task(
  subagent_type="gsd-project-researcher",
  model="{researcher_model}",
  team_name="gsd-research-{project_slug}",
  name="{stack|features|architecture|pitfalls}-researcher",
  description="{Dimension} research",
  prompt="
<role>
You are the {DIMENSION} researcher in a team researching {project_name}.
Your teammates: stack-researcher, features-researcher, architecture-researcher, pitfalls-researcher.
</role>

<your_task>
{Same task/question/downstream_consumer/quality_gate/output as the current Task() prompt for this dimension}
</your_task>

<team_collaboration>
After writing your research file:

1. Read your teammates' files if they exist (they may have started before you):
   - .planning/research/STACK.md (stack-researcher's output)
   - .planning/research/FEATURES.md (features-researcher's output)
   - .planning/research/ARCHITECTURE.md (architecture-researcher's output)
   - .planning/research/PITFALLS.md (pitfalls-researcher's output)

2. If you find a cross-dimension conflict or important dependency:
   SendMessage(type='message', recipient='{relevant-researcher}',
     content='Found something relevant to your research: {finding}',
     summary='Cross-dimension finding')

3. Update your own file if their findings change your recommendations.

4. Mark your task completed: TaskUpdate(taskId='{your_task_id}', status='completed')

5. SendMessage to the team lead:
   'RESEARCH COMPLETE: {dimension} done. File: .planning/research/{FILE}.md'
</team_collaboration>
" + (language_instruction ? "\n\n" + language_instruction : "")
)
```

**Note:** The `{task_id}` for each researcher is the ID returned by the corresponding
`TaskCreate` call in Step A. Pass it explicitly in the spawn prompt.

**Step C: Wait for all 4 tasks to complete**

Monitor via automatic message delivery. When all 4 researchers send "RESEARCH COMPLETE",
proceed.

**Step D: Shut down researcher teammates**

```
SendMessage(type="shutdown_request", recipient="stack-researcher",        content="Research complete. Thank you.")
SendMessage(type="shutdown_request", recipient="features-researcher",     content="Research complete. Thank you.")
SendMessage(type="shutdown_request", recipient="architecture-researcher", content="Research complete. Thank you.")
SendMessage(type="shutdown_request", recipient="pitfalls-researcher",     content="Research complete. Thank you.")
```

Wait for confirmations.

**Step E: Run synthesizer as subagent (same as current)**

```
Task(prompt="
<task>
Synthesize research outputs into SUMMARY.md.
</task>

<research_files>
Read these files:
- .planning/research/STACK.md
- .planning/research/FEATURES.md
- .planning/research/ARCHITECTURE.md
- .planning/research/PITFALLS.md
</research_files>

<output>
Write to: .planning/research/SUMMARY.md
Use template: ~/.claude/get-shit-done/templates/research-project/SUMMARY.md
Commit after writing.
</output>
" + (language_instruction ? "\n\n" + language_instruction : ""),
  subagent_type="gsd-research-synthesizer",
  model="{synthesizer_model}",
  description="Synthesize research"
)
```

**Step F: Clean up team**

```
TeamDelete()
```

---

### 5. `get-shit-done/workflows/new-milestone.md`

**Same change as new-project.md, in Step 8 "Research Decision".**

The existing research spawn block (4 × Task() + synthesizer) gets the same conditional wrapper:

```
**If `team_research` is false (default):** [existing behavior — unchanged]

**If `team_research` is true:** [same team-based flow as new-project.md above]
```

The project slug for the team name should use `current_milestone` from INIT:
`team_name="gsd-research-{current_milestone}"`.

Also:
- Parse `team_research` from INIT JSON (already added to `cmdInitNewMilestone` in step 2c)
- Add `team_research` to the "Extract from init JSON" list at the top of step 8

---

## Files That Do NOT Need Changes

| File | Reason |
|------|--------|
| `agents/gsd-project-researcher.md` | Agent is compatible as-is; team prompt wraps its role |
| `agents/gsd-research-synthesizer.md` | Runs as subagent, unchanged |
| `get-shit-done/workflows/research-phase.md` | Single-phase researcher, teams not needed |
| Any other workflow file | Team research is only for project/milestone initialization |
| `commands/gsd/new-project.md` | Routes to workflow, no changes needed |
| `commands/gsd/new-milestone.md` | Routes to workflow, no changes needed |

---

## How the System Works (Context for AI)

```
config.json
  └── "workflow": { "team_research": true }      ← stored here

gsd-tools.cjs cmdInitNewProject()
  └── returns team_research: true                 ← from loadConfig()

new-project.md step 6
  └── if team_research → TeamCreate + 4 Task(team_name=...) + TeamDelete
  └── else             → 4 Task() [existing, unchanged]

Researcher teammate flow:
  1. Writes .planning/research/{DIMENSION}.md
  2. Reads other researchers' files
  3. SendMessage to teammate if cross-dimension finding
  4. TaskUpdate → completed
  5. SendMessage to lead → "RESEARCH COMPLETE"

Lead flow:
  1. Waits for 4 "RESEARCH COMPLETE" messages
  2. Sends shutdown_request to all 4
  3. Spawns synthesizer subagent (unchanged)
  4. TeamDelete
```

---

## Verification Checklist After Implementation

- [ ] `config.json` template contains `"team_research": false`
- [ ] `gsd-tools.cjs` `loadConfig()` has `team_research` default and returns it
- [ ] `cmdInitNewProject` returns `team_research` in JSON
- [ ] `cmdInitNewMilestone` returns `team_research` in JSON
- [ ] `/gsd:settings` shows "Team Research" question (8th question)
- [ ] Default is "Off" — existing behavior preserved when `team_research: false`
- [ ] With `team_research: true`, running `/gsd:new-project` creates a `gsd-research-*` team
- [ ] 4 researcher tasks appear in task list during research
- [ ] Researchers can message each other (visible in UI)
- [ ] TeamDelete called after synthesizer completes
- [ ] `.planning/research/` files identical in quality to subagent approach
- [ ] Setting `team_research: false` reverts to old Task() behavior with no team created
