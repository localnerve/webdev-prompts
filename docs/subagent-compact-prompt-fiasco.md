# Subagent `/compact` Prompt Fiasco

This is a Claude exchange after Qwen 3.8 27b context window overruns and recovery attempt after a simple `/compact` request during progress on a refactor. Unexpectedly, the `/compact` command resulted in resource exhaustion and endless spawning of subagents, resulting in a total shutdown of the work.

## Conversation
This is the conversation in the aftermath to speculate root-cause and disccover how to prevent this from recurring.

## Question 1

> On my local AI, qwen 38 27b running in LMStudio using github copilot as the coding agent and session manager, I was doing a larger refactoring task that was going fine, stopped after the first two main modules, all going to plan. I noticed that there was quite a ways to go, and I was already at 160 messages. I had the agent save a session restore packet with a seed prompt, update repo memory, and stop. I then issued a /compact command. The result was the spawning of many subagents, slowing the system to a crawl and consuming almost all power and compute resources. I let it run for an hour, and the context window (262k) became exhausted. I tried it again, this time reminding it what I wanted, warned that I was running out of resources, and to try to compact the session again, but don't create any subagents. It proceeded to do it all again I gave it a half hour, then finally just killed the whole thing and unloaded the model. I'm currently rebuilding power and compute resources.
> How (and where - lmstudio server-side prompt or agent rules) can I create guard rails so this kind of thing is not possible again? I'd also like to create rules so that ANY task that requires subagents (I develop protocol.md based solutions that use subagents that I model locally then deploy on claude platform) only ever creates at most 2 subagents on my development computer. It should never create subagents for routine or maintenance work.

I'll check the current Copilot and LM Studio settings for this, since both have changed a lot recently.

The core problem is that a written rule is only a request, and your model ignored a direct instruction twice. Real guardrails have to remove the ability to spawn subagents or cap it at the server. Rules files are a useful second layer, not the main one.

### What probably happened

Compaction isn't a cheap operation. The summarization is a real LLM call on the same model the conversation uses, and the summary it builds includes objectives, the state of each file touched, and progress tracking of what's done versus partially done. At 160 messages near a 262k window, that is a very large prefill for a local 27B model.

The summary also contained your seed prompt and a "remaining work" plan, so the model likely treated it as an instruction to resume the refactor and fanned it out. That's a guess, but it fits the fact that it repeated even after you told it not to.

### Layer 1: LM Studio (hard cap on parallelism)

This is the closest thing to an enforceable "max 2" on your machine. Max Concurrent Predictions sets the maximum number of concurrent requests the model will process; requests beyond this limit are queued. By default it is set to 4. To change it, open the model loader, turn on "Manually choose model load parameters," then turn on "Show advanced settings."

- Set it to **2** for protocol development and **1** for routine work. Extra subagents then wait in line instead of all running at once.
- If you load from the CLI, `--parallel 1` (or 2) works, and you can confirm it with `lms ps`. Be aware of a reported bug: the CLI parallel flag drops K/V cache quantization, while the SDK path falls back to the default of 4. Always check `lms ps` after loading.
- Consider loading with a smaller context, such as 128k, for day-to-day work. That limits how big a single runaway can get.

Don't rely on an LM Studio preset system prompt for this. Copilot sends its own full system prompt with every request, and even when a server-side prompt gets through, it's still just text the model can ignore.

### Layer 2: Copilot (remove the tool)

**Custom agents** are the main lever. Each agent's frontmatter declares its tools. If you leave the tools list out, the agent inherits all tools. Spawning requires the `agent` tool set (you include it in `tools` to allow subagents), so an agent without it cannot spawn any.

Create two agents in `.github/agents/`.

`maintenance.agent.md`, your default for refactors, compaction and housekeeping:
```markdown
---
name: maintenance
description: Routine coding, refactoring, maintenance. Never delegates.
tools: ['read', 'search', 'edit', 'execute', 'todo']
---
You work alone. You have no ability to start subagents, and you must not
try to simulate them. Do one module at a time, then stop and report.
```

`protocol-dev.agent.md`, used only for testing your protocol.md solutions:
```markdown
---
name: protocol-dev
description: Local modeling of protocol.md multi-agent designs.
tools: ['read', 'search', 'edit', 'agent']
agents: ['worker-a', 'worker-b']
---
You may start at most 2 subagents in total for this task, and only from the
agents listed. Before starting one, state which one and why. Subagents may
not start their own subagents.
```

The `agents:` list is an allowlist, so this agent can only call the workers you name. Note that explicitly listing an agent there permits invoking it even if that worker has `disable-model-invocation: true`. Leave `agent` out of the workers' own `tools`.

**Tool picker:** in your normal chat mode, open the tools picker and uncheck the subagent tool (`runSubagent` / the agent set). Tool names vary by version, so match the frontmatter to what the picker shows.

**Settings** (`settings.json`). Search Settings for these, since names shift between releases:
- Keep nested subagents off (`chat.subagents.allowInvocationsFromSubagents`, which is off by default).
- `chat.agent.maxRequests`: set it low, such as 25–40. This caps total tool-call iterations per turn and is the closest thing to a limit on total subagent count rather than concurrent ones.
- Compact earlier so each compaction is cheaper. `chat.advanced.summarizeAgentConversationHistoryThreshold` accepts a ratio of the model's context window, such as 0.8 to compact at 80%. Try 0.5–0.6 for a local model.

### Layer 3: Written rules (soft, but still worth having)

Put these in `.github/copilot-instructions.md` (or `AGENTS.md`) so every agent sees them:

```markdown
### Resource rules (local dev machine)
- Never start subagents for routine work: refactoring, compaction,
  summarizing, memory updates, session save/restore, formatting, tests.
- Subagents are allowed only under the protocol-dev agent, max 2 per task,
  never nested.
- When compacting or summarizing, produce text only. Do not call tools.
  Do not resume work after a summary; wait for the user.
- If the user says to stop or not to delegate, comply immediately.
```

### Change the workflow

You had already done the right thing: you saved a restore packet, a seed prompt and the repo memory. **Skip `/compact` at that point and start a new chat** with the seed prompt under the `maintenance` agent. A fresh session costs almost nothing, while compacting 160 messages on local hardware is the most expensive thing you can ask for.

If you do compact, give it instructions. `/compact` accepts custom instructions after the command to guide the summary. For example: `/compact text summary only, no tool calls, do not continue the task`.

For a kill switch, use Stop in the chat panel first. If that doesn't work, `lms unload --all` frees the model without killing VS Code.

The LM Studio concurrency setting and an agent with no `agent` tool are the two changes that would have prevented this. Everything else just reduces how often you get near that point.

## Question 2

> This is the right idea, but remember I use local AI *with* remote AIs. So I model locally, sometimes doing full refactoring, but often the work ends up on Anthropic or OpenAIs services for bigger jobs. My point is the co-pilot instructions have to tell the agent to differentiate between endpoints. I want these restrictions *only* for LLMs with local endpoints. Also, provide a separate instruction or advise for non Github co-pilot, so your layer 2 and layer 3 solutions you presented are usable on other coding harnesses, not just Github co-pilot.

I'll verify the per-model and per-harness config options before writing this up.

You're right, and that changes the design. The model usually can't tell which endpoint it is running on. Copilot doesn't reliably tell it, and a local Qwen will happily guess. So one shared rule that says "if you're local, do X" will be applied inconsistently.

The fix is to tie the restrictions to whatever *selects* the local endpoint (an agent profile, a launch profile, or the server itself), not to the model's guess. Then add a written rule with a safe default for when that selection is ambiguous.

### The general pattern (any harness)

1. **The server cap only affects local work.** LM Studio's Max Concurrent Predictions applies only to requests sent to LM Studio. Anthropic and OpenAI jobs never touch it. This needs no endpoint detection at all, so set it to 2 and leave it there.
2. **Use a local profile and a remote profile.** Every harness has some way to bundle a model with tool permissions and instructions: agents in Copilot and OpenCode, profiles in Codex, settings plus environment in Claude Code. The local profile pins the LM Studio model *and* removes or limits the subagent tool. The remote profile is unrestricted.
3. **Make the profile state its endpoint.** Each profile's instructions include `ENDPOINT: LOCAL` or `ENDPOINT: REMOTE`. The shared rules file keys off that marker and treats a missing marker as LOCAL.
4. **Use a hard hook where the harness supports one.** A pre-tool hook can read the configured base URL and block or count subagent calls. This is the only fully reliable endpoint-aware limit.

### Shared rules file (portable)

Put this in `AGENTS.md` at the repo root. Copilot, Codex, OpenCode, Cursor and most others read it. For Claude Code, add a `CLAUDE.md` containing the line `@AGENTS.md` to import it.

```markdown
### Compute rules for local endpoints

#### Step 1: Determine your endpoint, in this order
1. Your active agent/profile instructions contain `ENDPOINT: LOCAL` or
   `ENDPOINT: REMOTE`. Use that.
2. Otherwise, if the base URL is visible and points to localhost,
   127.0.0.1, a LAN address, or port 1234/11434, or the provider is
   LM Studio / Ollama / llama.cpp / MLX: you are LOCAL.
3. Otherwise, if you cannot determine it: assume LOCAL.

#### Step 2: If LOCAL
- Never start subagents for routine or maintenance work: refactoring,
  compaction, summarizing, memory/repo-memory updates, session
  save/restore, formatting, linting, tests, docs.
- Subagents are allowed only when the task is explicitly modeling a
  protocol.md multi-agent design. Hard limit: 2 subagents total per task,
  never more than 2 at once, and subagents may never start subagents.
- Before starting a subagent, state its name, purpose, and count (1/2 or 2/2).
- When compacting or summarizing: text output only, no tool calls, and do
  not resume work afterward. Wait for the user.
- If the user says stop, or says not to delegate, comply immediately.
  Never re-plan around that instruction.

#### Step 2: If REMOTE
- No subagent limit from this file. Follow the task and protocol.md.
```

These rules are still soft. The structural pieces below enforce them.

### GitHub Copilot (VS Code)

Create agents in pairs, local and remote, and pin each one to a model in its frontmatter. In the local agents, the `agents` field is the hard control: use `[]` to prevent subagent use, or list only the two workers.

`.github/agents/local-maintenance.agent.md`:
```markdown
---
name: local-maintenance
description: Routine work on the LM Studio model. Never delegates.
model: <your LM Studio model name as shown in the picker>
tools: ['read', 'search', 'edit', 'execute', 'todo']
agents: []
---
ENDPOINT: LOCAL
You work alone. Do one module at a time, then stop and report.
```

`.github/agents/local-protocol-dev.agent.md`:
```markdown
---
name: local-protocol-dev
description: Local modeling of protocol.md multi-agent designs.
model: <your LM Studio model name>
tools: ['read', 'search', 'edit', 'agent']
agents: ['worker-a', 'worker-b']
---
ENDPOINT: LOCAL
Max 2 subagents per task, from the list only. No nesting.
```

Your remote agents (`remote-refactor.agent.md` and so on) set `model:` to the Claude or GPT model, include `ENDPOINT: REMOTE`, and keep `agents` open. The subagent tool is only removed in the local agents, so remote work is unaffected.

There is one gap to know about. If you're in a remote agent and switch the model picker to the local model by hand, the frontmatter limits of the local agent don't apply. Make it a habit to switch the agent, not just the model. LM Studio's concurrency cap is your backstop if you forget.

### Claude Code

This one can do real endpoint-aware enforcement. Point Claude Code at a local model through `ANTHROPIC_BASE_URL` (an Anthropic-compatible local endpoint or proxy). Then a hook can check that URL on every subagent call.

Subagents are gated by the `Agent` tool, and deny rules can target specific names like `Agent(Explore)`. PreToolUse hooks run before the permission system and can approve or deny a tool call in its place.

`.claude/settings.json`:
```json
{
  "hooks": {
    "PreToolUse": [
      { "matcher": "Agent|Task",
        "hooks": [{ "type": "command",
                    "command": "python3 .claude/hooks/local_subagent_cap.py" }] }
    ]
  }
}
```

`.claude/hooks/local_subagent_cap.py`:
```python
import json, os, re, sys, pathlib

MAX_LOCAL = 2
data = json.load(sys.stdin)
url = os.environ.get("ANTHROPIC_BASE_URL", "")
if not re.search(r"localhost|127\.0\.0\.1|\[::1\]|192\.168\.|10\.\d+\.|:1234|:11434", url):
    sys.exit(0)  # remote endpoint: no limit

counter = pathlib.Path(f"/tmp/cc_subagents_{data.get('session_id','x')}")
n = int(counter.read_text()) if counter.exists() else 0
if n >= MAX_LOCAL:
    print(f"Blocked: local endpoint subagent limit ({MAX_LOCAL}) reached.", file=sys.stderr)
    sys.exit(2)  # exit 2 blocks the call and shows the reason to the model
counter.write_text(str(n + 1))
```

This hard-enforces "max 2 per session, local only." To block subagents completely during maintenance, start a separate local session with `"deny": ["Agent"]` in `.claude/settings.local.json`. Also be aware of reports that subagents don't inherit the parent's settings and hooks, so keep the cap at the parent level. It stops the third spawn whatever the children do.

### OpenCode

Permissions include `task`, which controls launching subagents and matches on the subagent type. Agent-level permissions override the global config. Make a local agent with the LM Studio model and either `task: deny` or an allowlist:

```markdown
---
description: Local protocol modeling
model: lmstudio/<model-id>
permission:
  task:
    "*": deny
    worker-a: allow
    worker-b: allow
---
ENDPOINT: LOCAL
```

The allowlist limits *which* subagents can run. The count of 2 comes from the written rule plus LM Studio's cap.

### Codex CLI

Use profiles. You can create local and cloud profiles in `config.toml` and switch with `codex --profile local` or `codex --profile cloud`. The local profile uses `model_provider = "lmstudio"`.

Codex has no `task`-style permission to deny, so here you rely on `AGENTS.md`, a local profile that loads the `ENDPOINT: LOCAL` instructions, and the LM Studio cap. Check your version's docs for a per-profile instructions setting. If there isn't one, start the local profile from a small shell alias that also sets an environment marker the rules file can mention.

### Cline, Roo, Continue and others

Most of these have per-mode or per-profile model selection and some form of tool allowlist. Apply the same pattern: create a local mode with no delegation tool, give it the `ENDPOINT: LOCAL` marker, and share the `AGENTS.md` rules. Wherever a harness can't remove the tool, LM Studio's concurrency setting is still enforcing the limit.

Of all of these, the Claude Code hook is the only true "max 2, local only" counter. Everywhere else, the profile that removes the tool and the server cap carry the weight, and the rules file covers the rest.