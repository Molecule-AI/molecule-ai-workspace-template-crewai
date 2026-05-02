# CLAUDE.md — molecule-ai-workspace-template-crewai

Workspace template that runs CrewAI inside the Molecule AI workspace runtime.
This is a **single-process container** booted by `molecule-runtime` (PyPI:
`molecule-ai-workspace-runtime`); it speaks A2A to peer workspaces and exposes
canvas chat through the platform.

## What this template does

- Boots a CrewAI `Agent` + `Task` + `Crew` per inbound message.
- Wires platform tools (delegation, memory, sandbox, approval, MCP bridges)
  into the Crew's tool list so the agent can call them like any CrewAI tool.
- Speaks the A2A protocol to other workspaces — replies and peer delegations
  flow over the platform's A2A transport, not direct HTTP.

## Files

- `adapter.py` — the CrewAI adapter (only Python you usually edit).
- `config.yaml` — runtime config: model, env requirements, model picker entries.
- `system-prompt.md` — default backstory; overridden by canvas Config tab post-deploy.
- `Dockerfile` — base image; `ENTRYPOINT ["molecule-runtime"]` boots the runtime
  which discovers `Adapter` via `ADAPTER_MODULE=adapter`.
- `requirements.txt` — `molecule-ai-workspace-runtime` + `crewai>=0.100.0`.

## BaseAdapter integration point

`adapter.py` defines `CrewAIAdapter(BaseAdapter)` from
`molecule_runtime.adapters.base`. Two lifecycle hooks matter:

- `setup(config: AdapterConfig)` — imports `crewai`, calls
  `self._common_setup(config)` (inherited; loads skills, builds LangChain tool
  list, resolves the system prompt), then bridges each LangChain tool to a
  CrewAI `@tool` via `_langchain_to_crewai()`.
- `create_executor(config) -> AgentExecutor` — returns a `CrewAIA2AExecutor`
  that the runtime hands to the A2A server. **The executor is the contract**;
  do not call CrewAI from `setup()`.

Module exports `Adapter = CrewAIAdapter` so `molecule-runtime` finds it.

## CrewAI specifics

`CrewAIA2AExecutor.execute(context, event_queue)` does the per-message work:

1. `extract_message_text(context)` — pull the user/peer message off A2A.
2. `set_current_task(heartbeat, brief_task(...))` — heartbeat surfaces the
   current task to the platform UI; cleared in `finally`.
3. Build a fresh `Agent(role, goal, backstory, llm, tools)`,
   `Task(description, expected_output, agent)`, and `Crew([agent], [task])`.
4. `await asyncio.to_thread(crew.kickoff)` — CrewAI is sync; off-load to a
   thread so the event loop stays responsive for heartbeats + A2A.
5. `event_queue.enqueue_event(new_text_message(reply))` — A2A reply.

`role` is the first 100 chars of the resolved system prompt; `goal` is fixed.
History from the A2A context is folded into `task_desc` via `build_task_text`.

Model strings follow Molecule's `provider:model` form (e.g.
`openai:gpt-4.1-mini`). `openai:` is rewritten to `openai/` for CrewAI's
LiteLLM model router. Other providers pass through unchanged.

## Tool wrapping (Molecule MCP -> CrewAI)

`_langchain_to_crewai(lc_tool)` wraps each LangChain `BaseTool` (the platform
tool surface — `delegate_task`, `send_message_to_user`, memory, sandbox, etc.)
as a sync CrewAI `@tool`. CrewAI's decorator reads `__doc__` at decoration
time, so `wrapper.__doc__` is set from the LangChain `description` **before**
applying `crewai_tool(...)`. The wrapper bridges sync->async via
`asyncio.get_event_loop().run_until_complete(lc_tool.ainvoke(kwargs))`.

## Common gotchas

- **Sync/async boundary.** `crew.kickoff()` is blocking; always wrap with
  `asyncio.to_thread`. The tool wrapper uses `run_until_complete`, which works
  because CrewAI invokes tools from worker threads, not the main loop.
- **CrewAI tool docstrings are mandatory.** Setting `__doc__` after the
  `@tool` decorator runs is too late — keep the order in `_langchain_to_crewai`.
- **Model provider routing.** Only `openai:` prefix rewrite exists today;
  Anthropic / Bedrock / others rely on LiteLLM defaults from the raw string.
- **Per-message Crew construction.** A new `Agent`/`Crew` is built every call
  — there is no in-memory CrewAI state across messages. State lives in the
  history extracted from A2A context, not inside CrewAI.
- **Errors swallowed to a reply.** The `try/except` in `execute()` returns
  `f"CrewAI error: {e}"` to the user instead of raising; check container logs
  for stack traces.

## Conventions

- New Molecule platform tools land in `molecule-core` / runtime — they will
  show up automatically once `_common_setup` returns them in `langchain_tools`.
  Do not register tools by hand here.
- Skills load via `_common_setup` driven by `config.yaml` `skills:` (none in
  this template by default).
- `/workspace` is the agent's read/write scratch dir; `/configs/config.yaml`
  is the live config mount.
- Logs: stdout/stderr from this process is captured by the platform; use the
  module logger (`logger = logging.getLogger(__name__)`).

## What NOT to do

- Don't break the `BaseAdapter` contract (`setup` + `create_executor`) — the
  runtime discovers and drives it, and the publish-runtime smoke test boots
  this adapter from a freshly built wheel.
- Don't bypass A2A for inter-workspace calls. Use the `delegate_task` tool;
  direct HTTP between workspaces will skip auth, audit, and the platform's
  A2A queue.
- Don't import CrewAI at module top level — `setup()` does the import inside
  a `try/except ImportError` so missing deps fail with a clear message.
- Don't edit code paths assumed by the runtime smoke test without updating
  the corresponding template-publish workflow in `.github/workflows/`.
