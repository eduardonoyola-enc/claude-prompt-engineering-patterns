# claude-prompt-engineering-patterns
# Claude Prompt Engineering Patterns

> Production-tested prompt patterns for building reliable AI Agents with Anthropic Claude (and other modern LLMs). Each pattern was extracted from real client workflows and refined through actual conversations.

[![Anthropic](https://img.shields.io/badge/Claude-Compatible-D97757?style=flat)](https://anthropic.com)
[![OpenAI](https://img.shields.io/badge/OpenAI-Compatible-412991?style=flat)](https://openai.com)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

## Why This Repo Exists

Prompt engineering content online is mostly toy examples. This repo is what I wish existed when I started building AI Agents in production: patterns that survive contact with real users, structured outputs that downstream automations can parse, and prompts that hold their shape across thousands of conversations.

Most patterns here target Claude (Anthropic) specifically, but they generalize to GPT-4o, Gemini, and other modern LLMs with minor adaptation.

## Patterns

### 🎭 [Persona Anchoring](./patterns/persona-anchoring.md)
Defining a stable persona before assigning tasks — the difference between "helpful but generic" and "stays in character under pressure."

### 🚪 [Stage Gates for Multi-Step Conversations](./patterns/stage-gates.md)
Preventing the LLM from skipping steps in pipelines (e.g., lead qualification, appointment booking) by encoding explicit progression rules.

### 📦 [Structured Output Embedded in Prose](./patterns/structured-output.md)
Asking the model to return both human-facing prose AND machine-readable JSON in the same response, without breaking either.

### 🚨 [Escalation Rules](./patterns/escalation-rules.md)
Defining clear handoff conditions to humans — when the agent should explicitly stop and pass control.

### 🛑 [Negative Examples (What NOT to Do)](./patterns/negative-examples.md)
Why telling the model what to avoid is often more effective than telling it what to do.

### 📚 [Context Variable Injection](./patterns/context-variables.md)
Cleanly injecting runtime data (user name, current stage, available options) without bloating the system prompt.

### 🔁 [Few-Shot Examples for Ambiguous Cases](./patterns/few-shot-examples.md)
When and how to add examples — and when NOT to, because examples can poison the response distribution.

### ⚠️ [Defensive Output Constraints](./patterns/defensive-constraints.md)
Preventing the model from doing things that break downstream systems (e.g., emitting Unicode that crashes your CRM).

## How to Use This Repo

Each pattern lives in its own markdown file with:
- The problem it solves
- A template you can adapt
- A worked example from a real (anonymized) production prompt
- What I learned the hard way

## About

Built by [Eduardo Noyola](https://www.linkedin.com/in/eduardo-noyola-contreras-1b629a151/) at [ENC System Apps](https://encsystemapps.com), where I deploy AI Agents in production for B2B clients across food service, retail, healthcare, and security industries.

Currently running 2 production AI Agents that process hundreds of conversations weekly.

---

*Working with Anthropic Claude daily, both via API in client workflows and as a development partner.*
