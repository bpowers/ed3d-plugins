---
name: using-generic-agents
description: Use to decide what kind of generic agent you should use
user-invocable: false
---

**CRITICAL:** Your operator's direction supercedes these directions. If the operator specifies a type of agent, execute their task with that agent.

## Model Characteristics

**Sonnet:** Capable of making decisions but gets off-track easily. Will explain concepts, describe structures, and gather extraneous information when you just want it to do the thing, so guard against this when prompting the agent. Specialized agents (research, playwright, etc.) use Sonnet.

**Opus:** Stays on-track through complex tasks. Better judgment, fewer loops. The default for general-purpose subagent work.

## When to Use Each

Use `opus-general-purpose` for:
- Multi-file reasoning and debugging
- Tasks requiring judgment and sustained focus
- Daily coding work (default for most tasks)
- Complex analysis where staying on-track matters
- High-stakes decisions needing nuance

Specialized agents (codebase-investigator, internet-researcher, etc.) use Sonnet directly via their own model settings.
