# AI Alarm Escalation — tiered notify with acknowledgment and timeout

An n8n workflow for **industrial / IoT alarm management** — not an alert
notifier. The difference is the whole point: a notifier sends a message and
hopes someone sees it. This escalates until a human *acknowledges*, suppresses
the alarm flood that would otherwise wake someone fifty times, routes by
severity, and keeps an audit trail of every outcome.

Import `ai-alarm-escalation-workflow.json` into any n8n instance to inspect it.
It runs with no credentials for inspection; wire the notification channel and
(optionally) the Claude triage key to run it live.

## Why this isn't a basic "send an alert" flow

Anyone can wire a webhook to an SMS node. Real alarm handling — the kind a
SCADA or process-control environment actually needs — requires five things
that a notifier skips, and this template implements each:

### 1. Flood / chatter suppression
A flapping sensor can re-fire the same alarm dozens of times a minute. The
workflow remembers each alarm's last-seen time (keyed by source + alarm ID) in
workflow static data and suppresses repeats inside a configurable window, so
one fault pages a person **once**, not fifty times. Suppressed alarms are still
logged.

### 2. Tiered escalation with acknowledgment and timeout
This is the core. A critical alarm notifies the on-call person with an
acknowledgment link, then **waits**. If no acknowledgment arrives within the
timeout, it escalates to the backup, waits again, then to the manager. The
`Wait` nodes resume on either the acknowledgment webhook *or* the timeout —
the ACK-or-escalate loop that separates an escalation system from a notifier.

### 3. Severity-based routing
Critical alarms run the full escalation chain. Warnings notify once, quietly.
Info-level alarms are logged only — nobody is paged. Routing by severity is
basic alarm-management discipline and keeps the escalation chain reserved for
what actually matters.

### 4. AI triage that stays advisory
Claude *summarizes* the alarm and suggests a likely cause and priority — useful
context for whoever gets paged. It **never** decides whether to suppress an
alarm or whom to notify; deterministic rules do that. The triage call is
non-blocking: if Claude is unconfigured or down, alarms still escalate. "AI
advises, rules decide" is the correct posture for anything safety-adjacent.

### 5. Audit trail
Every outcome — suppressed, acknowledged (and by which tier), escalated to the
manager, warning-notified, info-logged — lands at a `Log:` node with a
timestamp, ready to wire to Google Sheets or a database. When an alarm system
matters, you need to be able to answer "who was told, when, and did they
respond?"

## The flow

```
Alarm webhook → normalize → flood suppression
     ├─ suppressed → log and stop
     └─ not suppressed → Claude triage (advisory) → severity route
            ├─ critical → Tier 1 notify → wait for ACK
            │                 ├─ ACK → log, done
            │                 └─ timeout → Tier 2 notify → wait for ACK
            │                        ├─ ACK → log, done
            │                        └─ timeout → Tier 3 (manager) → log
            ├─ warning  → notify once → log
            └─ info     → log only
```

## Setup to run it live

1. **Notification channel** — replace the `Notify` nodes' placeholder URL with
   your SMS / Slack / Teams / email endpoint. Keep the `ackUrl` field
   (`$execution.resumeUrl`) so recipients can acknowledge.
2. **Acknowledgment** — the recipient opens the `ackUrl`, or your system POSTs
   `{ "ack": true }` to it. No ACK before the timeout escalates to the next tier.
3. **Claude triage (optional)** — add a Header Auth credential (`x-api-key` =
   your Anthropic key) to the triage node. Leave it unconfigured and alarms
   still escalate; you just lose the AI summary.
4. **Audit log** — wire the `Log:` nodes to Google Sheets or a database.
5. **Tune** — the suppression window (default 5 min) and the ACK timeout
   (default 5 min per tier) are set on the Flood Suppression and Wait nodes.

## Note

Sized and labeled as a generic, importable template. Node `typeVersion`s and
the `Wait`/ACK condition may need a small adjustment to match your n8n version —
import it, run a test alarm through each severity, and confirm the ACK-or-timeout
branch resumes the way you expect before relying on it.
