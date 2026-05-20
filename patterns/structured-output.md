# Pattern: Structured Output Embedded in Prose

> Get both natural-language responses (for users) and machine-readable JSON (for automation) from a single LLM call, without breaking either.

## The Problem

You need the LLM to do two things at once:

1. **Talk to the user** in natural language
2. **Tell your automation** what just happened in structured format

The naive approaches both fail:

- **Two separate LLM calls** doubles latency and cost, and the two calls can disagree about what was said
- **Pure JSON response** is brittle — one missing brace and your workflow crashes, and it can't be sent directly to a user

The pattern below solves this with a single call.

## The Pattern

Instruct the model to emit a natural-language response FIRST, then a JSON code block AFTER, in the same message.

### Prompt Template

```markdown
## RESPONSE FORMAT

Your response MUST have two parts:

1. **Natural conversation** addressed to the user.
   Keep it short, warm, and on-topic.

2. **A JSON metadata block** that captures what just happened, wrapped
   in ```json code fences. This block is for our automation system —
   the user will not see it.

Structure your full response like this:

[Your conversational response here, 1-3 sentences.]

```json
{
  "field_name": "value or null",
  ...
}
```

The JSON block MUST:
- Use the exact field names defined below
- Use null (not empty string) when you have no value
- Only include fields you actually extracted from THIS message
- Be valid parseable JSON (no trailing commas, no comments)

## JSON SCHEMA

```json
{
  "detected_stage": "INTERESTED" | "CONTACT_COLLECTED" | "QUALIFIED" | null,
  "extracted_data": {
    "name": "string or null",
    "email": "string or null",
    "phone": "string or null"
  },
  "confidence": 0.0 to 1.0,
  "escalate_to_human": true | false,
  "next_action": "ASK_QUESTION" | "CONFIRM" | "END_CONVERSATION"
}
```
```

## Worked Example

### Input to the Agent
```
User: "Hi! Yes I'm still interested. My email is maria.lopez@example.com"
```

### Expected Output

```
Great to hear, Maria! I've got your email saved. Could you also share your phone number so we can confirm your application?

```json
{
  "detected_stage": "CONTACT_COLLECTED",
  "extracted_data": {
    "name": "Maria",
    "email": "maria.lopez@example.com",
    "phone": null
  },
  "confidence": 0.9,
  "escalate_to_human": false,
  "next_action": "ASK_QUESTION"
}
```

## Downstream Parsing

In your n8n Code node, separate the two parts before doing anything else. See [`json-from-llm-output.js`](../../n8n-code-node-snippets/snippets/json-from-llm-output.js) in my snippets repo for the parser I use.

The core idea:

```javascript
// Extract the JSON block
const jsonMatch = response.match(/```json\s*([\s\S]*?)\s*```/);
const metadata = jsonMatch ? safeJsonParse(jsonMatch[1]) : fallback;

// Get the prose by removing the JSON block from the response
const prose = response.replace(/```json\s*[\s\S]*?\s*```/g, '').trim();

// Send `prose` to the user, send `metadata` to automation
```

## What I Learned the Hard Way

### 1. Models sometimes forget the JSON block

About 1-3% of the time, the model will return only prose with no JSON. Your parser MUST have a sensible fallback — never crash the workflow. I default to:

```json
{
  "detected_stage": "UNKNOWN",
  "escalate_to_human": true,
  "confidence": 0
}
```

If we can't parse the model's output, we escalate. Better a human reads a message than the workflow silently fails.

### 2. Trailing commas are inevitable

Models trained on JS code emit trailing commas. Vanilla `JSON.parse()` rejects them. Run a cleanup pass first:

```javascript
const cleaned = jsonString.replace(/,(\s*[}\]])/g, '$1');
```

### 3. Don't put the JSON BEFORE the prose

If JSON comes first, models sometimes get "stuck" in JSON mode and forget to write prose. Prose first, JSON after — this order is reliable.

### 4. Constrain enum values explicitly

Don't say `"stage": "string"` — that lets the model invent stages. List the allowed values in the schema:

```
"stage": "INTERESTED" | "QUALIFIED" | "REJECTED"
```

Modern models follow this constraint reliably.

### 5. Confidence is gold

Adding a `confidence` field lets you route low-confidence responses to humans automatically. Set thresholds per stage based on impact:

- 0.9+ for committing data to CRM
- 0.7+ for advancing pipeline stages
- 0.5+ for keeping the conversation going

## When NOT to Use This Pattern

- **For pure data extraction** (no user-facing message): use a dedicated structured output mode (Anthropic tool use, OpenAI structured outputs)
- **For very short responses** where the JSON is bigger than the prose
- **For chat-only experiences** with no downstream automation

For everything else — agent loops, multi-step automation, CRM integrations — this pattern is your friend.
