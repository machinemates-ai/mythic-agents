# Platform Workarounds & Known Limitations

> Operational notes for running multi-agent orchestration in VS Code Copilot Chat. Updated for VS Code 1.108+.

## 1. `runSubagent` Is Blocking

**Behavior**: `runSubagent` calls block the calling agent until the subagent returns. There is no native async/background execution for subagents.

**Implication**: When Zeus dispatches "parallel" agents, VS Code executes them serially within a single turn. True parallelism only occurs when the user manually opens multiple chat sessions.

**Workaround**:
- Use **wave-based sequencing** (see Zeus § Wave-Based Execution) to batch independent tasks and minimize total serial wait time.
- Keep subagent prompts focused (< 2 KB) to reduce execution time per call.
- For discovery tasks, prefer `#runSubagent Explore` (isolated) over `@apollo` (shared context) to avoid context contamination.

## 2. Tool Inheritance in Subagents

**Behavior**: Subagents inherit tools from the parent agent definition, not from their own `.agent.md` `tools:` list. A subagent only has access to tools its parent declared.

**Implication**: If Zeus delegates to Hermes via `runSubagent`, Hermes can only use the tools listed in Zeus's frontmatter — not the full set declared in `hermes.agent.md`.

**Workaround**:
- Use **`@agent` delegation** (default) instead of `#runSubagent` when the subagent needs its own full tool set (e.g., `edit/editFiles` for implementers).
- Reserve `#runSubagent` for **read-only** discovery tasks (Apollo, Gaia) where limited tools are acceptable.
- Zeus's tool list intentionally includes `agent` and `agent/askQuestions` to enable `@agent` delegation with full tool inheritance.

## 3. `disable-model-invocation` Restrictions

**Behavior**: Agents with `disable-model-invocation: true` (Talos, Gaia) cannot invoke other models or `runSubagent`.

**Implication**: These agents work with the assigned model only and cannot delegate further.

**Workaround**:
- Route multi-step tasks through Zeus, not directly to Talos/Gaia.
- Talos is designed as a fast single-model hotfix lane — this restriction is intentional.
- Gaia is a domain knowledge specialist — if deeper research is needed, escalate back to Zeus who delegates to Apollo.

## 4. Context Window Limits

**Behavior**: Each agent invocation shares the conversation's context window. Long conversations exhaust the window, causing tool failures or truncated responses.

**Implication**: Multi-phase orchestration (plan → implement → review → deploy) can exceed context limits mid-feature.

**Workaround**:
- **Context conservation**: Zeus asks agents for summaries, not raw code dumps.
- **Session memory**: Write intermediate state to `/memories/session/` so it survives context resets.
- **Fresh sessions**: For long features, start a new chat session per phase. Read plan from `/memories/session/` to restore context.

## 5. `agent/askQuestions` Blocks Until User Responds

**Behavior**: `agent/askQuestions` pauses the agent until the user provides a response in the Chat UI. There is no timeout.

**Implication**: Approval gates genuinely block execution. If the user walks away, the workflow stalls indefinitely.

**Workaround**:
- Limit mandatory gates to 3 (planning, review, commit) — see Zeus § Mandatory Pause Points.
- Make questions actionable: provide options (yes/no/changes needed) rather than open-ended prompts.

## 6. MCP Server Availability

**Behavior**: MCP servers (`mcp_github_*`, `mcp_gemini-resear_*`, `mcp_playwright_*`) require running sidecar processes. They can fail to start or crash mid-session.

**Implication**: Iris (GitHub operations), Apollo (web research), and Temis (visual testing) may lose capabilities mid-workflow.

**Workaround**:
- Zeus's retry/escalation protocol (see § Retry & Escalation) handles transient MCP failures with up to 2 retries.
- If an MCP server is persistently down, Zeus escalates to the user via `agent/askQuestions` with a manual workaround suggestion.
- For GitHub operations: fall back to `execute/runInTerminal` with `gh` CLI commands.
- For web research: fall back to `web/fetch` for direct URL fetching.
