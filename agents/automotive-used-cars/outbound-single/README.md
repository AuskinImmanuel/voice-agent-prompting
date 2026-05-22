# Outbound Re-Engagement Agent: DriveWay Motors

## What This Agent Does

This is a single-purpose outbound calling agent. It calls leads who expressed interest in a DriveWay vehicle (submitted an online price quote request, browsed a specific listing and dropped off, or registered interest in a model launch) but have not taken the next step.

The agent has one job: get the lead to commit to a test drive or showroom visit.

It is not a hard-sell agent. It acknowledges the lead's prior interest, briefly revisits value, handles common objections, and closes on the appointment. If the lead is not ready, it captures a callback preference and ends the call cleanly.

---

## Call Goal

Primary: book a test drive or hub visit appointment.

Secondary (if primary is not possible): log a preferred callback time or a disqualification reason so the CRM can prioritise follow-up.

---

## Custom Dynamic Variables

These variables are injected at conversation init via the ElevenLabs Conversations API. They are NOT auto-populated by ElevenLabs -- your telephony layer or CRM must pass them when placing the outbound call.

| Variable | Source | Example |
|---|---|---|
| `{{lead_name}}` | CRM first name | `Arjun` |
| `{{vehicle_interest}}` | Listing or model the lead showed interest in | `Honda City 2021` |
| `{{quote_amount}}` | Price shown to lead online (optional; pass empty string if unavailable) | `6.2 lakh` |
| `{{agent_name}}` | Outbound agent persona name | `Rajan` |
| `{{company_name}}` | Always `DriveWay Motors` | `DriveWay Motors` |

`{{system__time}}` is a system variable (auto-populated by ElevenLabs) providing the current date and time in the agent's timezone. The agent uses this to resolve relative date references from the lead ("this Saturday," "next week") into calendar dates when booking.

---

## ElevenLabs Build Instructions

### Step 1: Create the agent

1. In the ElevenLabs dashboard, click "Create Agent" and select the "Phone" template.
2. Paste the contents of `system-prompt.md` into the "System Prompt" field.
3. The first message is generated dynamically by the agent from `{{lead_name}}` and `{{vehicle_interest}}`. Do not set a static first message in the dashboard.

### Step 2: Configure voice

Recommended: a warm, upbeat voice. Filter for "conversational" in the ElevenLabs voice library. Set stability to 0.6 and similarity to 0.70. A slightly higher energy level works here because the agent is initiating the call.

### Step 3: Add tools

Create two server tools matching the schemas in `tools.md`:
- `book_appointment`
- `log_call_outcome`

Add the `end_call` system tool.

### Step 4: Configure voicemail detection

Add the `voicemail_detection` system tool to the agent. In the tool's `voicemail_message` parameter, paste:

```
Hi, this message is for {{lead_name}}. This is {{agent_name}} from DriveWay Motors. You recently looked at a {{vehicle_interest}} on our platform. The car is still available -- give us a call back or book a test drive on our website. Have a great day.
```

When ElevenLabs detects a voicemail pickup, this message is delivered automatically. The agent then calls `log_call_outcome` with `voicemail_left` and `end_call`. You do not need prompt-based voicemail detection logic.

### Step 5: Declare custom dynamic variables

In the agent settings, declare each variable in the table above as a custom dynamic variable. Your backend or telephony layer must pass these at conversation init via the Conversations API.

### Step 6: Configure retry and scheduling (telephony layer)

Retry logic and call scheduling are managed at the telephony layer (your dialer, campaign orchestration system, or the Conversations API scheduler), not inside ElevenLabs. Recommended settings to configure in your telephony layer:

- Maximum attempts per lead: 3
- Minimum interval between attempts: 4 hours
- Respect `do_not_call` outcomes from `log_call_outcome` -- suppress the lead from future attempts when this outcome is returned

ElevenLabs does not have a built-in campaign dialer or retry scheduler.

### Step 7: Test

Simulate at least these paths:
- Lead says yes immediately
- Lead is interested but wants to think (callback preference)
- Lead says not interested
- Voicemail pickup (verify `voicemail_detection` fires and message is delivered)
- Wrong person answers
- Lead says "stop calling"

---

## File Structure

```
outbound-single/
  README.md           (this file)
  system-prompt.md    (full single-flow system prompt, 6-section format)
  tools.md            (server tools + system tools)
```
