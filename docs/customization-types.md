# Customization types, harness-agnostic

Every coding-agent harness (GitHub Copilot, Claude Code, Continue, Cline, Kilo Code, Codex,
...) lets you shape the model with Markdown files, but each uses its own names and folders.
Underneath, there are four kinds. The useful question for each is **who triggers it** and
**when it enters the context window**.

| Kind (this repo's name) | Triggered by | Loads | Typical size | Folder here |
|---|---|---|---|---|
| **Rule** | The harness, automatically | Every turn (optionally scoped by file glob) | Tiny: a few lines | `rules/` |
| **Prompt** (command / workflow) | You, by name (`/name`) | Only on the turn you invoke it | Any | `prompts/` |
| **Skill** | The model, when a task matches the skill's description | Description always; body on demand | Medium, can bundle scripts | `skills/<name>/SKILL.md` |
| **Agent** (persona / mode / subagent) | You select it, or a parent agent delegates to it | Replaces or wraps the system prompt for the whole session | Medium | `agents/` |

Rules and prompts are about *what text gets injected*. Skills are about *letting the model
pull in text itself*. Agents are different in kind: an agent is a **whole configuration**:
instructions **plus** a tool allowlist, a model choice, and (optionally) which other agents
it may delegate to.

## What each harness calls them

| Kind | GitHub Copilot (VS Code / CLI) | Claude Code | Continue.dev | Cline | Kilo Code |
|---|---|---|---|---|---|
| Rule | "custom instructions": `.github/copilot-instructions.md`, `.github/instructions/*.instructions.md` | `CLAUDE.md` | `.continue/rules/*.md` | `.clinerules/` | rules in `kilo.jsonc` / `.kilo/rules-{mode}/` |
| Prompt | "prompt files": `.github/prompts/*.prompt.md` | slash commands: `.claude/commands/*.md` | `.continue/prompts/*.prompt.md` | "workflows": `.clinerules/workflows/*.md` | `.kilo/commands/*.md` |
| Skill | "agent skills": `.github/skills/`, `.agents/skills/` (personal: `~/.copilot/skills/`, `~/.agents/skills/`) | `.claude/skills/` (personal: `~/.claude/skills/`) | check current docs | check current docs | check current docs |
| Agent | **"custom agents"**: `.github/agents/*.agent.md` (personal: `~/.copilot/agents/`) | **"subagents"**: `.claude/agents/*.md` (personal: `~/.claude/agents/`) | "agents" (config-based) | no custom personas (Plan/Act modes only) | "custom modes" |

`AGENTS.md` at a repo root is read by many harnesses (Copilot, Codex, and others) and acts
as a **rule**, despite the name. It doesn't define an agent.

Cells marked "check current docs" are areas that have moved quickly; verify against your
installed version before relying on them.

## Portability notes

**Skills are the most portable.** `SKILL.md` follows an open standard (agentskills.io), so
the same folder works unchanged across skills-compatible harnesses; only the parent folder
differs. `.agents/skills/` is the most vendor-neutral location where it's supported.

**Agents are the least portable.** The Markdown body (behavior, workflow, constraints)
moves across harnesses as-is. The YAML frontmatter does not:

| Frontmatter field | Portable? | Notes |
|---|---|---|
| `name`, `description` | Yes | Same meaning everywhere that has agents. The description is what a parent agent reads when deciding to delegate. |
| `tools` | **No** | Tool names are harness-specific (Copilot: `read`, `edit`, `execute`, `web/fetch`, `server/*`; Claude Code: `Read`, `Edit`, `Bash`, `WebFetch`, `mcp__server__*`). MCP server names also depend on your local config. Translate per harness. |
| `model` | **No** | Model identifiers differ per harness and per provider setup. |
| `agents` (Copilot) | Partly | Which subagents this agent may delegate to. Claude Code expresses delegation differently (any subagent is callable by description). |
| `argument-hint`, `handoffs` (Copilot) | No | UI affordances; drop when porting. |

So in this repo, an agent file is stored in Copilot's `.agent.md` format (the source it was
written in), and the catalog in the README notes what has to change to run it elsewhere.

## Choosing a kind

- Must the model *always* know it? → **rule** (keep it tiny; it costs context every turn).
- Do *you* decide when it's needed? → **prompt**.
- Should the *model* decide when it's needed, from the task? → **skill**.
- Does it need different tools, a different model, or isolation from the main
  conversation? → **agent**.
