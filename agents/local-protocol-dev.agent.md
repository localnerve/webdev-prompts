---
name: local-protocol-dev
description: Local modeling of protocol.md multi-agent designs.
argument-hint: A repo task to model subagent work (Qwen3.8-27B local).
agents: ['worker-a', 'worker-b']
tools: ['read', 'search', 'edit', 'execute', 'todo', 'web/fetch', 'GitKraken/*', 'context7/*']
---

<!-- Tip: Use /create-agent in chat to generate content with agent assistance -->

ENDPOINT: LOCAL

Max 2 subagents per task, from the list only. No nesting.