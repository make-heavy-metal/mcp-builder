# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This repo **is both a Claude Code plugin and a single-plugin marketplace**. The same root directory serves two roles:
- `.claude-plugin/plugin.json` → the `mcp-builder` plugin manifest
- `.claude-plugin/marketplace.json` → the `make-heavy-metal` marketplace catalog (lists this plugin with `"source": "./"`)

There is nothing to compile, build, or deploy. The plugin packages Anthropic's MCP-server-building guide so other Claude Code sessions can use it. Users install via `/plugin marketplace add make-heavy-metal/mcp-builder` then `/plugin install mcp-builder@make-heavy-metal`.

Future plugins under Make Heavy Metal will be added as additional entries in `marketplace.json`, most likely sourced from sibling repos via `{"source": "github", "repo": "make-heavy-metal/<name>"}` rather than crammed into this repo.

When a user asks you to "build an MCP server" or similar, the `mcp-builder` skill's `SKILL.md` is the authoritative process. Load reference files on demand as that document instructs — do not preload them.

## Layout

```
.
├── .claude-plugin/
│   └── plugin.json                   # Plugin manifest (name, version, author)
├── skills/
│   ├── mcp-builder/                  # Auto-activating skill — the 4-phase guide
│   │   ├── SKILL.md                  # Entrypoint with trigger phrases in description
│   │   └── references/               # Load on demand per SKILL.md instructions
│   │       ├── mcp_best_practices.md # Universal guidelines (naming, pagination, transport, security)
│   │       ├── node_mcp_server.md    # TypeScript/MCP SDK patterns — preferred stack
│   │       ├── python_mcp_server.md  # Python/FastMCP patterns
│   │       └── evaluation.md         # How to author the eval XML file
│   └── evaluate/
│       └── SKILL.md                  # User-invoked /mcp-builder:evaluate slash command
├── scripts/                          # Referenced by both skills via ${CLAUDE_PLUGIN_ROOT}/scripts
│   ├── evaluation.py                 # Eval harness: runs QA XML through Claude with server's tools attached
│   ├── connections.py                # stdio / SSE / streamable-HTTP client adapters
│   ├── example_evaluation.xml        # QA-pair XML format reference
│   └── requirements.txt              # anthropic>=0.39.0, mcp>=1.1.0
├── LICENSE, README.md, CLAUDE.md
```

The four reference files and the main SKILL.md form a progressive-disclosure system: `SKILL.md` has the process and links; the agent loads deeper docs only when the current phase needs them. **Preserve this structure when editing — don't inline reference content into `SKILL.md`.**

## Running the Evaluation Harness

Inside Claude Code, invoke `/mcp-builder:evaluate <eval.xml> <server-description>` — the `evaluate` skill at `skills/evaluate/SKILL.md` parses the natural-language server description and builds the right `evaluation.py` invocation.

To run the harness directly (outside the plugin flow):

```bash
pip install -r scripts/requirements.txt
export ANTHROPIC_API_KEY=...

# stdio server (harness launches it)
python scripts/evaluation.py -t stdio -c python -a path/to/server.py eval.xml

# streamable HTTP (server already running)
python scripts/evaluation.py -t http -u https://example.com/mcp -H "Authorization: Bearer TOKEN" eval.xml

# write report to file instead of stdout
python scripts/evaluation.py -t stdio -c node -a dist/index.js eval.xml -o report.md
```

Default model is `claude-3-7-sonnet-20250219` (hardcoded in `evaluation.py:223`); override with `-m`. Scoring is exact string match between `<response>` tags and `<answer>` in the QA XML — questions must therefore produce a single canonical value (number, ID, exact string).

## Editing Guidance

- **Stack preference in the guide is TypeScript**, not Python (see `SKILL.md` §1.3). When the user asks which SDK to recommend for a new server, default to TypeScript unless they say otherwise.
- **Transport preference is streamable HTTP (remote) or stdio (local)**. SSE is deprecated per `mcp_best_practices.md` — don't recommend it for new servers even though the eval harness still supports it for backward compatibility.
- **Naming conventions are load-bearing** and enforced by the guide: Python servers use `{service}_mcp`, Node uses `{service}-mcp-server`, tools use `{service}_{action}_{resource}` snake_case. Don't silently deviate when drafting examples.
- **Path conventions inside the plugin:** the `evaluate` skill uses `${CLAUDE_PLUGIN_ROOT}/scripts/...` to resolve paths — this is the Claude Code plugin convention. Don't hardcode absolute paths.
- When updating `SKILL.md`, keep the phase structure (Research → Implementation → Review/Test → Evaluation) intact — the eval harness and reference docs are indexed against these phases.
- When updating `SKILL.md`'s frontmatter `description`, keep the trigger phrases ("build an MCP server", "create an MCP server for X", etc.) — they drive skill auto-activation.
