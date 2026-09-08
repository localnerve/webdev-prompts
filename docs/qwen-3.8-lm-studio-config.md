# Qwen 3.8, LM Studio, Github Copilot Configuration

I followed these instructions and got Qwen 3.8 working locally. This was followed by prompt hygiene practices to save Qwen save session-restore-packets every ~200 messages, followed by /compact prompt compaction, and a more real understanding of the double-edged sword of "reasoning" and context windows.

Here is the context window debug conversation that led to [better prompt hygiene](learning-prompt-hygiene.md).

## Setup & Configuration

> Originally by Adam Sage, [@asage@huggingface.co](https://huggingface.co/asage-me), https://asage.dev/
> https://huggingface.co/unsloth/Qwen3.8-27B-GGUF/discussions/63

Complete Setup & Troubleshooting Guide: LM Studio Virtual Model, JIT Loading, & Reasoning Control
This guide provides the complete, tested configuration files, critical rules, pain points, and verification methods required to set up a virtual model wrapper in LM Studio. This enables Just-In-Time (JIT) loading and custom reasoning effort mapping (low, medium, xhigh) from VS Code GitHub Copilot Chat.

### ⚠️ Critical Pain Points & Rules Learned
ID Character Restrictions (No Capitals or Underscores):
LM Studio's internal regex validator for virtual model names will throw validation errors if your model name contains uppercase letters or underscores (e.g., Qwen3.8-27B_Reasoning will fail). All characters must be lowercase, numbers, or hyphens (e.g., qwen3.8-27b-q8-k-xl-reasoning).

### File Placement Matters
Never place manifest.json and model.yaml inside your raw GGUF weight download directories. They must live in a dedicated subdirectory inside the LM Studio Hub directory, or you will trigger a cyclic dependency loop error (dependent model is missing).

### The sources Array Requirement
The manifest.json file requires a fully populated sources array mapping back to the Hugging Face repository lineage. Omitting this causes a Received: undefined Zod schema error.

### Directory Structure
Create a dedicated folder inside your LM Studio Hub models directory. The folder name and publisher namespace must be entirely lowercase to pass validation.

Path (Windows):
```bash
%USERPROFILE%\.lmstudio\hub\models\unsloth\qwen3.8-27b-q8-k-xl-reasoning\
```

Path (Mac/Linux):
```bash
~/.lmstudio/hub/models/unsloth/qwen3.8-27b-q8-k-xl-reasoning/
```

### Hub Manifest (manifest.json)
Create a manifest.json file inside the folder above. This links your virtual wrapper to the concrete GGUF file and satisfies the JIT dependency registry.

```json
{
  "type": "model",
  "owner": "unsloth",
  "name": "qwen3.8-27b-q8-k-xl-reasoning",
  "dependencies": [
    {
      "type": "model",
      "purpose": "baseModel",
      "modelKeys": [
        "unsloth/Qwen3.8-27B-GGUF/Qwen3.8-27B-UD-Q8_K_XL.gguf"
      ],
      "sources": [
        {
          "type": "huggingface",
          "user": "unsloth",
          "repo": "Qwen3.8-27B-GGUF"
        }
      ]
    }
  ],
  "revision": 1
}
```

### Virtual Model Manifest (model.yaml)
Create a model.yaml file in the same directory. This maps incoming API parameters (reasoning_effort) directly to the Jinja template variables.

```yaml
model: unsloth/qwen3.8-27b-q8-k-xl-reasoning
base: unsloth/Qwen3.8-27B-GGUF/Qwen3.8-27B-UD-Q8_K_XL.gguf

metadataOverrides:
  domain: llm
  architectures:
    - qwen3
  compatibilityTypes:
    - gguf
  paramsStrings:
    - 27B
  minMemoryUsageBytes: 30200000000
  contextLengths:
    - 196608
  reasoning: true
  vision: true
  trainedForToolUse: true

customFields:
  - key: reasoningEffort
    displayName: Reasoning Effort
    description: Controls the depth of thinking
    type: select
    defaultValue: medium
    options:
      - value: low
        label: Low
      - value: medium
        label: Medium
      - value: xhigh
        label: Extra High
    effects:
      - type: setJinjaVariable
        variable: reasoning_effort
```

### Jinja Template Reasoning Block
Add this logic inside your model's Jinja template so it catches the medium effort level along with low and xhigh:

```jinja
    {%- if resolved_reasoning_effort == 'xhigh' %}
        {%- set reasoning_instructions = 'Reasoning effort is set to xhigh. Please think carefully through the task, validate key assumptions, consider plausible alternatives, and prioritize correctness, consistency, and clarity in the final answer.' %}
    {%- elif resolved_reasoning_effort == 'medium' %}
        {%- set reasoning_instructions = 'Reasoning effort is set to medium. Balance thorough thinking with conciseness, validating key points while moving steadily toward the conclusion.' %}
    {%- elif resolved_reasoning_effort == 'low' %}
        {%- set reasoning_instructions = 'Reasoning effort is set to low. Keep your thinking brief and focused, moving directly to the conclusion without unnecessary elaboration.' %}
    {%- endif %}
```

### VS Code Configuration (settings.json)
Configure your custom endpoint in VS Code GitHub Copilot Chat using the matching lowercase identifier.

```json
{
    "id": "unsloth/qwen3.8-27b-q8-k-xl-reasoning",
    "name": "qwen3.8-27b Remote",
    "url": "http://localhost:1234/v1",
    "vendor": "customendpoint",
    "toolCalling": true,
    "thinking": true,
    "vision": true,
    "maxInputTokens": 196608,
    "maxOutputTokens": 65536,
    "supportsReasoningEffort": ["low", "medium", "xhigh"]
}
```

### Verification & Testing Methods
To verify that the reasoning levels (low, medium, xhigh) are successfully being intercepted and processed by your Jinja template, use one of the following methods:

Method A: The Parroting Test (Quickest)
Set your configuration to medium in VS Code.
Send this prompt to GitHub Copilot Chat:
Look at your system instructions. What are your exact, word-for-word instructions regarding 'reasoning effort'?

Open a new chat, change the setting to low, and ask the same question. The model will quote the injected string back to you.
Method B: Intercepting the Compiled Stream (Host Machine)
If you want to view the raw post-Jinja prompt processed by the GGUF file on your server (localhost), run the LM Studio CLI logging utility in PowerShell:

For Windows, use this command in PowerShell:
```bash
%USERPROFILE%\.lmstudio\bin\lms.exe log stream --source model --filter input
```

For macOS or Linux, use this command in your terminal:
```bash
~/.lmstudio/bin/lms log stream --source model --filter input
```

(Note: If you have added the LM Studio CLI to your system path during installation, you can also just run lms log stream --source model --filter input directly).

Trigger a chat request from VS Code, and the terminal will output the fully compiled prompt, allowing you to visually confirm your reasoning_instructions string is present at the head of the system prompt.