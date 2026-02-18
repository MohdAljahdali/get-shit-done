---
name: gsd:team-execute-phase
description: Execute a phase using an agent team — persistent executor teammates coordinate via shared task list, enabling direct inter-agent communication and parallel self-coordination
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - Grep
  - Glob
  - Task
  - AskUserQuestion
  - TeamCreate
  - TeamDelete
  - TaskCreate
  - TaskGet
  - TaskUpdate
  - TaskList
  - SendMessage
---

<objective>
Execute all plans in a phase using a Claude Code agent team.

Unlike `/gsd:execute-phase` which spawns fire-and-forget subagents, this command creates
a persistent team of executor teammates that:
- Coordinate through a shared task list
- Self-claim work as dependencies resolve between waves
- Communicate directly with each other and the lead
- Allow the user to interact with individual teammates during execution

Best used for phases with multiple plans across several waves where parallel execution
and inter-agent coordination add real value.
</objective>

<execution_context>
@~/.claude/get-shit-done/workflows/team-execute-phase.md
</execution_context>

<process>
Follow the team-execute-phase workflow from `@~/.claude/get-shit-done/workflows/team-execute-phase.md`.
</process>
