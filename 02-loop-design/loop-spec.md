# Loop Spec: Cortex

## Trigger and loop type

Heartbeat (cadence)

## Why this loop type

Run it every 3 hours starting 9 am UK time till 6 pm UK time each work day (excluding Saturday, Sunday and public holidays)

The heartbeat *wakes* Cortex on this cadence to check for inbound PM tasks; each task it finds then runs to completion (draft → propose → stop for PM review) before the loop idles until the next beat.

## Definition of done

A summary highlighting the status of the project, stories completed, stories in the backlog, team velocity, success stories and challenges

## Stop conditions

- **Success**: When draft is created
- **Stuck / give up**: No progress after 3 iterations
- **Escalate to human**: When reports needs to be sent or the issues or challenges are more than 5

## State

Handled tasks, sprint backlog, project A

## The five components

- **Work tree**: Keep it for each project - so use it for single JIRA instance
- **Skills**: get-status-update, get-summary
- **Plugins / connectors**: JIRA, Monday.com, emails (outlook), teams group chat
- **Subagents**: Norms-check, character count, tone check
- **State tracking**: Handled tasks, sprint backlog, project A

## Context plan

Write and compress

## Hand-off to bounds and evals (M5)

Never exceed $1 and never run more than 3 iterations

