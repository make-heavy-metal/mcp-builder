# mcp-builder

A Claude Code plugin that packages Anthropic's guide for building high-quality MCP (Model Context Protocol) servers, plus an evaluation harness for scoring them against QA-pair test suites.

## What's in the plugin

- **`mcp-builder` skill** — Anthropic's 4-phase workflow (research → implement → test → evaluate) with per-language references for TypeScript (recommended) and Python. Activates automatically on prompts like *"build an MCP server for X"*, *"wrap the Y API as MCP tools"*, or *"write an MCP server in TypeScript"*.
- **`/mcp-builder:evaluate` slash command** — Wraps `scripts/evaluation.py`. Takes a QA XML file plus a server description (stdio command, http URL, or sse URL) and returns a scored markdown report.

## Install

**Per-user (local development):** symlink this repo into your user plugins directory so it's available in every session:

```bash
mkdir -p ~/.claude/plugins
ln -s "$(pwd)" ~/.claude/plugins/mcp-builder
```

**Per-project:** symlink or copy the repo into a project's `.claude/plugins/` directory. Claude Code auto-discovers plugins in both locations on startup.

**From a marketplace:** once published, install via `/plugin install mcp-builder@<marketplace>`.

## Requirements

For the evaluation harness only:

```bash
pip install -r scripts/requirements.txt   # anthropic>=0.39.0, mcp>=1.1.0
export ANTHROPIC_API_KEY=...
```

The skill itself has no runtime dependencies — it's a guide the agent reads.

## Usage

**Build an MCP server** — just describe what you want. The skill auto-activates:

> *"Build an MCP server in TypeScript that exposes the Linear API"*

**Evaluate a server** — invoke the slash command with an eval file and a server description:

```
/mcp-builder:evaluate my-eval.xml stdio: node dist/index.js
/mcp-builder:evaluate my-eval.xml http://localhost:3000/mcp with header Authorization: Bearer $TOKEN
```

See `scripts/example_evaluation.xml` for the QA-pair format and `skills/mcp-builder/references/evaluation.md` for how to author good eval questions.

## License

See `LICENSE`. Original content authored by Anthropic; packaged as a plugin by Travis Corrigan.
