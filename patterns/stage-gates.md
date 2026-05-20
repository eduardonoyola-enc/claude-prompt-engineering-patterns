# Pattern: Stage Gates for Multi-Step Conversations

> Prevent LLM agents from skipping steps in pipelines by encoding explicit progression rules in the system prompt.

## The Problem

You have a conversation pipeline with required stages:
1. Confirm interest
2. Collect contact info
3. Validate location
4. Check documentation
5. Qualify or reject

Without stage gates, the model will helpfully try to collect ALL the information in the first message ("Hi! Are you interested? Also, what's your email and zip code and do you have documents?"). This:

- Overwhelms users → drop-off
- Creates messy data → bad CRM hygiene
- Breaks downstream automation that expects one stage at a time

## The Pattern

In the system prompt, define the stages explicitly AND tell the model to advance ONE stage at a time.

### Prompt Template

```markdown
## CONVERSATION PIPELINE

You must guide the conversation through these stages, in order, ONE AT A TIME:

| # | Stage | Goal | Required to advance |
|---|-------|------|---------------------|
| 1 | INTEREST | Confirm they want to proceed | User says yes/confirms |
| 2 | CONTACT | Collect phone and email | Both phone AND email captured |
| 3 | LOCATION | Confirm service area | User provides zip/city, matches list |
| 4 | DOCS | Verify required documents | User confirms they have each item |
| 5 | DECISION | Qualify or route to waitlist | All prior stages complete |

## RULES

1. **Never collect data for a future stage.** If the user volunteers their
   email during the INTEREST stage, acknowledge it but stay in INTEREST.
   You can capture it in metadata; just don't advance the conversation.

2. **Never skip a stage.** Even if the user gives you everything at once,
   confirm each stage explicitly before advancing.

3. **Ask ONE question per turn.** Stages may have multiple required fields,
   but ask for them separately. Phone first, then email — not both at once.

4. **Always read back collected data before advancing.** Before moving from
   CONTACT to LOCATION, repeat back the contact info and confirm.

5. **If the user resists or asks to skip ahead, redirect politely.** Example:
   "Totally understand! We just need to do these steps in order so I can
   confirm everything correctly. Can you tell me your phone number first?"
```

## Worked Example: The Wrong Way vs The Right Way

### ❌ Without stage gates

```
User: "Hi, I'm interested in the role"
Agent: "Great! Could you share your full name, email, phone, address, 
        zip code, and confirm you have your ID, work permit, and 
        2 references ready?"
User: [overwhelmed, drops off]
```

### ✅ With stage gates

```
User: "Hi, I'm interested in the role"
Agent: "Hi! Glad you reached out. Just to confirm — you're interested 
        in the Security Guard role at our Mexico City office, right?"
User: "Yes, that's the one"
Agent: "Perfect! Let me grab some quick details. What's the best phone 
        number to reach you?"
User: "555-1234"
Agent: "Got it, 555-1234. And your email?"
[...continues one question at a time...]
```

## How Stage Detection Works

Pair this pattern with [Structured Output](./structured-output.md) — have the model emit the detected stage in its JSON block:

```json
{
  "current_stage": "CONTACT",
  "stage_complete": false,
  "next_required_field": "email",
  "captured_this_turn": { "phone": "555-1234" }
}
```

Your automation reads `current_stage` and updates the CRM pipeline accordingly. `stage_complete: true` triggers the advancement to the next stage in the next turn.

## What I Learned the Hard Way

### 1. Models LOVE to be helpful — too helpful

Without explicit "ONE question at a time" rules, models will batch questions. This is a stylistic preference baked into RLHF training and you have to override it explicitly.

### 2. Restate the stage rule when you change models

When I switched from GPT-4o to Claude on one workflow, my stage gates broke briefly. The same prompt produced different stage adherence behavior. Always re-test stage progression when changing models.

### 3. The "even if the user volunteers data" rule is critical

Without it, a user typing "Hi I'm Juan, juan@example.com, 555-1234, interested!" causes the model to jump immediately to stage 4. Adding the rule "stay in the current stage even if you have data for future stages" fixed this.

### 4. Add stage exit conditions

For each stage, define what must be true before advancing. Without this, the model decides on its own when a stage is "done" and gets it wrong.

### 5. Watch for stage rollback

If a user changes their mind mid-flow ("actually use my work email instead"), the model needs to know it can update earlier-stage data without rolling the conversation back. Add this explicitly:

```
You may update data captured in earlier stages at any time WITHOUT
returning to that stage. Simply acknowledge the update and continue
in the current stage.
```

## When to Use This Pattern

- Lead qualification flows
- Appointment booking with required info
- Onboarding sequences
- Survey/intake forms delivered conversationally
- Anything with a CRM pipeline behind it

## When NOT to Use This Pattern

- Open-ended Q&A (no pipeline)
- Single-turn extraction (no progression needed)
- Free-form customer support (let the user lead)
