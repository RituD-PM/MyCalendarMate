# CalendarMate — AI-Powered Productivity Assistant

Capstone submission for *Applied Agentic AI for PMs/TPMs* — a five-workflow, multi-agent n8n system built for ServionIQ's PMs/TPMs, addressing the ~2 hours/day of scheduling, calendar-checking, and email-triage overhead described in the problem statement.

## Architecture

**Pattern: Hierarchical delegation.** A single Orchestrator classifies each chat message and delegates to exactly one specialist sub-workflow via n8n's Tool Workflow mechanism, then returns that specialist's result to the user.

```
User (n8n Chat Trigger)
    -> Orchestrator Agent (gpt-5-mini)
         -> Briefing Agent Tool     -> Briefing Agent Workflow
         -> Scheduler Agent Tool    -> Scheduler Agent Workflow
         -> Email Agent Tool        -> Email Agent Workflow
         -> Follow-Up Agent Tool    -> Follow-Up Agent Workflow
```

## Files in this repo

| File | Purpose |
|---|---|
| `Orchestrator.json` | Entry point: chat trigger, intent routing, conversation memory, the four tool-workflow calls |
| `Briefing_Agent_Workflow.json` | Daily briefing: Calendar + Gmail read, conflict detection, grounded summary |
| `Scheduler_Agent__Workflow.json` | Availability check, slot proposal/conflict handling, event creation with a Meet link |
| `Email_Agent_Workflow.json` | Unread-inbox digest, grouped by urgency/topic |
| `Follow-Up_Agent_Workflow.json` | Meeting recap + action-item extraction, conditional send to attendees |

## Setup

1. **Import all five workflow JSON files** into your n8n instance (`Import from File` for each).
2. **Rebind credentials** on every node that has one — the credential IDs in these files are specific to the original instance and won't resolve in a new one. You'll need:
   - A **Google Calendar OAuth2** credential (used in Briefing and Scheduler)
   - A **Gmail OAuth2** credential (used in Briefing, Email, and Follow-Up)
   - An **OpenAI API key** credential (used by every `OpenAI Chat Model` node, running `gpt-5-mini`)
3. **Re-point the Orchestrator's four Tool nodes at your own imported workflow IDs.** Each `toolWorkflow` node (`Briefing Agent Tool`, `Scheduler Agent Tool`, `Email Agent Tool`, `Follow-Up Agent Tool`) references a `workflowId` that must match the ID n8n assigns to your imported copy of that sub-workflow — these will differ from the ones in this export.
4. **Replace the hardcoded personal email address** in `Briefing_Agent_Workflow.json` → `Prepare Params` (`userEmail`) and → `Get Today's Events` (`calendar`) with your own address, or better, parameterize it per user before any multi-user rollout.
5. **Activate all five workflows**, then open the Orchestrator's chat interface to test.

## Known limitations (current state, as of this export)

These are real, currently-unaddressed gaps found by direct testing during development — noted here so they aren't mistaken for solved:

- **No observability/tracing is wired up.** The Orchestrator's output currently connects to nothing (`"Orchestrator Agent": {"main": [[]]}`). This does **not** satisfy the assignment's observability requirement as-is — connect a LangSmith or Langfuse integration, or at minimum reinstate a trace-logging step, before treating evaluation as complete.
- **Scheduler cannot update an existing calendar event** — only create. There is no event-ID tracking across turns, so a "fix the time on that meeting" follow-up request has no workflow path to succeed.
- **No per-user authorization boundary.** All five workflows currently point at one person's Google account; this only works for a single-user pilot, not the 50-user rollout target, until each user's requests are scoped to their own calendar/inbox.
 
