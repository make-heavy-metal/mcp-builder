---
name: evaluate
description: Run the MCP evaluation harness against a user's MCP server using a QA-pair XML file. Wraps scripts/evaluation.py (stdio/http/sse transports) and returns a scored markdown report. Invoke with `/mcp-builder:evaluate <eval.xml> <server-description>` — server-description can be natural language describing how to launch or reach the server.
argument-hint: <eval.xml> <how to reach the server — e.g. "stdio: node dist/index.js" or "http://localhost:3000/mcp with header X: Y">
allowed-tools: Bash, Read, Write
---

# /mcp-builder:evaluate

You are wrapping `${CLAUDE_PLUGIN_ROOT}/scripts/evaluation.py` for the user. They have provided a QA XML file and a description of their MCP server. Your job: parse the description, build the right invocation, run it, and summarize the report.

## Procedure

1. **Verify prerequisites.** Run these in parallel:
   - `test -f "${CLAUDE_PLUGIN_ROOT}/scripts/evaluation.py" && echo OK` (script exists)
   - `python3 -c "import anthropic, mcp" 2>&1` (deps installed)
   - `test -n "$ANTHROPIC_API_KEY" && echo OK || echo MISSING` (API key set)

   If deps are missing, tell the user to run:
   ```
   pip install -r "${CLAUDE_PLUGIN_ROOT}/scripts/requirements.txt"
   ```
   If `ANTHROPIC_API_KEY` is unset, stop and ask the user to export it before retrying. Do not prompt them for the key value.

2. **Parse the user's arguments.** The first argument is the path to the eval XML file. Everything else describes the server. Map the description to transport flags:

   | User says | Transport | Flags |
   |---|---|---|
   | "stdio: node dist/index.js" | `stdio` | `-t stdio -c node -a dist/index.js` |
   | "stdio: python server.py with API_KEY=xyz" | `stdio` | `-t stdio -c python -a server.py -e API_KEY=xyz` |
   | "http://host/mcp" or "https://..." | `http` | `-t http -u <url>` |
   | "http://host/mcp with header Authorization: Bearer X" | `http` | `-t http -u <url> -H "Authorization: Bearer X"` |
   | "sse://..." or "SSE endpoint ..." | `sse` | `-t sse -u <url>` |

   When the user's description is ambiguous, ask one targeted clarifying question rather than guessing.

3. **Verify the eval file exists** and preview it:
   ```bash
   test -f "<eval.xml>" && head -30 "<eval.xml>"
   ```
   If it's missing, stop and tell the user the path didn't resolve.

4. **Run the evaluation.** Allocate a tempfile with `mktemp` (so concurrent invocations don't clobber each other), then write the report there:
   ```bash
   REPORT=$(mktemp -t mcp-eval-report.XXXXXX.md)
   python3 "${CLAUDE_PLUGIN_ROOT}/scripts/evaluation.py" \
     <flags-from-step-2> \
     "<eval.xml>" \
     -o "$REPORT"
   echo "Report: $REPORT"
   ```
   Stream with a long timeout (evals take minutes). If the user passed `-m <model>` in their arguments, forward it; otherwise accept the script default.

5. **Summarize the report.** Read `$REPORT` and produce:
   - Accuracy (correct / total)
   - Average duration and tool calls per task
   - Names of any failed tasks and a one-line reason each (from their `<feedback>` block)
   - Path to the full report

   Do **not** paste the whole report — it's long. Offer to show specific failed tasks if the user asks.

## Notes

- The harness is Python 3 and must find `mcp` + `anthropic` on the active interpreter. If the user's system has multiple Pythons, use `python3` explicitly.
- For stdio transport, the harness launches the server itself — never start it separately first.
- For http/sse, the user must start their server before invoking this skill.
- Scoring is exact string match on `<response>` tags. A "failed" task may just mean the answer format differs — surface the actual vs. expected so the user can judge.
