# Pattern: Defensive Output Constraints

> Prevent the LLM from producing output that breaks downstream systems.

## The Problem

LLMs don't know about your downstream APIs' quirks. They'll happily emit:
- Emojis that crash your CRM's text parser
- Markdown formatting that your SMS gateway strips
- URLs that get rewritten by link trackers
- Long messages that exceed channel limits
- Characters that need escaping in your database

These don't show up in development. They show up in production at scale, where you find out 8 hours later that 200 messages got truncated.

## The Pattern

Add explicit "DO NOT" rules to the system prompt for known production constraints. Pair with [snippets/unicode-sanitizer.js](../../n8n-code-node-snippets/snippets/unicode-sanitizer.js) as a defense-in-depth measure.

### Prompt Template

```markdown
## OUTPUT CONSTRAINTS

You are sending messages through systems with known limitations.
Follow these rules to prevent message corruption:

### Emoji rules
- DO NOT use these emojis (they contain Unicode variation selectors
  that break our CRM's text dispatch):
  ✉️ ⭐ 🗺️ ☀️ ✏️ 🛠️ 🔧 ⚙️ ⚠️ ❤️
  
- USE these alternatives instead:
  📧 (email) · 🌟 (star) · 🌍 (location) · 🌞 (sun) · 📝 (notes)
  🛠 (no FE0F) · ⚡ (lightning) · 🚨 (alert) · 💙 (heart)

### URL rules
- DO NOT use https:// prefixes in URLs (our message system rewrites
  them, often breaking message length limits)
- USE www.example.com format instead

### Length rules
- Keep messages under 1,000 characters total
- Keep individual paragraphs under 200 characters
- Prefer multiple short messages over one long one

### Formatting rules
- DO NOT use markdown (** bold **, * italics *, # headers)
- DO NOT use code blocks with ``` for normal communication
- USE plain text formatting: line breaks for emphasis, ALL CAPS sparingly

### Character rules
- DO NOT use tab characters (some channels render as spaces, breaking alignment)
- DO NOT use smart quotes (" " ' ') — use straight quotes (" ')
- DO NOT use em-dashes (—) — use double hyphens (--) or commas
```

## Worked Example

### ❌ Without defensive constraints

```
Agent response:
"Welcome! ✉️ Here's everything you need to know:
**Step 1:** Visit https://www.ourservice.com/onboarding
**Step 2:** Watch the video at https://video.ourservice.com/intro
**Step 3:** Reply with your availability and we'll get started! ⭐"

What downstream systems see:
"Welcome!" (truncated at FE0F after the envelope)
```

### ✅ With defensive constraints

```
Agent response:
"Welcome! 📧 Here's everything you need:

Step 1 - Visit www.ourservice.com/onboarding
Step 2 - Watch the intro video at www.video.ourservice.com/intro
Step 3 - Reply with your availability and we'll get started! 🌟"

What downstream systems see:
The full message, intact.
```

## What I Learned the Hard Way

### 1. Constraint drift over conversation length

In long conversations, the model starts forgetting the constraints. The longer the context, the more likely a banned emoji slips through.

**Fix:** Re-inject the most critical constraints into the prompt every N turns, or always sanitize the output before sending. I do both.

### 2. Constraints stack — and conflict

When you add too many constraints, the model gets paralyzed and produces overly cautious, robotic messages. Pick your top 5 most critical constraints, defend them aggressively, and let smaller issues be handled by a post-processing sanitizer.

### 3. Use both prompt-level and code-level defenses

NEVER rely only on the LLM to follow constraints. Always sanitize the output server-side too. The prompt makes constraint violations rare; the sanitizer makes them impossible.

### 4. Test with adversarial inputs

Ask testers to try to break your agent's output:
- Ask it to "send you an example with emojis"
- Ask for a "formal letter with headers"
- Ask for "all the links to your services"

These probes will surface constraint violations you'd miss in happy-path testing.

### 5. Document WHY in the prompt

"DO NOT use ✉️" makes the model curious about exceptions. "DO NOT use ✉️ because it contains a Unicode variation selector (FE0F) that breaks our CRM's text dispatch" is treated as a hard rule.

Modern models respect reasoned constraints more than arbitrary ones.

## Constraint Categories to Always Define

1. **Character/encoding** — what bytes will break your downstream parsers
2. **Length** — what your channels can deliver
3. **Format** — what your renderers will display correctly
4. **Content** — what you legally/ethically should never say (PII, claims, etc.)
5. **Style** — what matches your brand voice

Treat constraints as part of the prompt's structure, not an afterthought.
