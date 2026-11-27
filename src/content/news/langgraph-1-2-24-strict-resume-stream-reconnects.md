---
title: "LangGraph 1.2.16 to 1.2.24: strict resume, streams that rejoin after a restart, and deploy commands scripts can read"
description: "A catch-up on LangGraph releases from Oct 16 to Nov 20, 2026: a strict resume mode that refuses stale interrupt answers, SDK streams that retry and rejoin after a server restart, error handlers that recover on every path, faster reads on long threads, and JSON output for deploy list and deploy metrics."
date: 2026-11-24T08:30:00Z
playerSlug: "langgraph"
sourceName: "LangGraph GitHub releases"
sourceUrl: "https://github.com/langchain-ai/langgraph/releases/tag/1.2.24"
category: "Release"
tags:
  - orchestration
  - human-in-the-loop
  - streaming
  - deployment
  - open-source
draft: false
ogImage: "/images/players/langgraph/langgraph-editorial.webp"
---

LangGraph, LangChain's MIT-licensed framework for stateful agent graphs, shipped nine core releases between Oct 16 and Nov 20, 2026 (`1.2.16` to `1.2.24`), along with Python SDK `0.4.8` to `0.4.14`, CLI `0.4.36` to `0.4.43`, checkpoint `4.3.1` and `4.3.2`, checkpoint-postgres `3.2.1` and prebuilt `1.1.2` and `1.1.3`. This note covers the user-facing changes from `langchain-ai/langgraph`. Dependency bumps and most small fixes are omitted.

What a human gets: less time lost to runs that silently went wrong. With strict resume on, a stale answer to an interrupt fails with an error instead of being ignored, a stream cut by a server restart picks up again, and deploy commands return JSON a script can read.

![Conceptual editorial illustration of a branching agent graph with a loop back to an earlier node, not a product screenshot.](/images/players/langgraph/langgraph-editorial.webp)

*Editorial illustration from the [LangGraph player page](/players/langgraph/), not a test of the features below.*

## Human answers that do not get lost

**Strict resume.** Before `1.2.20` (Nov 5), a `Command(resume=...)` that named an interrupt that was no longer pending was ignored: the graph paused on the same interrupts again with no error. An opt-in strict mode now raises `UnknownInterruptError` before writing anything when a resume names an interrupt that is not pending, is an empty map, or is one value while several interrupts wait ([PR 9574](https://github.com/langchain-ai/langgraph/pull/9574)). Without the setting, nothing changes.

**One id per interrupt.** In the same release, each `interrupt()` call in a task gets its own id, so an answer retried after a node asked its next question is not taken as the answer to that question. A node that fails after getting an answer and is retried reuses the answer instead of asking again ([PR 9589](https://github.com/langchain-ai/langgraph/pull/9589)).

**From the SDK.** Python SDK `0.4.14` (Nov 6) adds `strict_resume=True` to `runs.create`, `stream` and `wait`. A resume that answers an interrupt that is not waiting, for example one already answered in another tab, fails the run and leaves the thread as it was ([PR 9599](https://github.com/langchain-ai/langgraph/pull/9599)). A server that does not support it ignores the field.

## Streams that survive a restart

**Reconnects back off.** Python SDK `0.4.11` (Oct 30) retries a stream reconnect that fails to open, such as a 429, a 5xx or a transport error while the server restarts or deploys, with waits of 1, 2, 4, 5 and 5 seconds, instead of ending the stream ([PR 9462](https://github.com/langchain-ai/langgraph/pull/9462)). `runs.wait`, `join` and `cancel(wait=True)` reconnects retry with backoff too.

**Cut streams rejoin.** Python SDK `0.4.12` (Nov 3) rejoins a run stream that closes before the run ends, for example on a proxy timeout or a server restart. Before, the caller got a cut answer and no error. The client now checks the run's status once and reconnects while it is still pending or running ([PR 9554](https://github.com/langchain-ai/langgraph/pull/9554)).

## Fewer runs that fail or crawl

**Error handlers recover on every path.** Core `1.2.18` (Oct 26) fixes a node's `error_handler` running but the original error still being raised when streaming `messages` or `custom`, with `subgraphs=True`, or next to a parallel node ([PR 9408](https://github.com/langchain-ai/langgraph/pull/9408)).

**Faster reads on long threads.** The same release makes `DeltaChannel` take a full snapshot every 100 updates instead of 1,000. The release notes call it faster reads on long threads for more storage, and `snapshot_frequency=1000` keeps the old behavior ([PR 9420](https://github.com/langchain-ai/langgraph/pull/9420)). SDK `0.4.13` (Nov 5) adds `select` to `threads.get_state` and `get_history`, so the Agent Server can skip loading state keys you do not need. The PR measured about 295 ms down to 35 ms on a 300-turn thread with `select=["values.messages"]` ([PR 9568](https://github.com/langchain-ai/langgraph/pull/9568)).

**Postgres and tool fixes.** checkpoint-postgres `3.2.1` (Nov 14) skips the async saver's lock for connection pools and runs a lone store read outside a pipeline. Core `1.2.23` and prebuilt `1.1.3` (Nov 19) give tool calls dispatched with `Send` the same state as an edge, and prebuilt `1.1.3` makes `wrap_tool_call` state overrides reach the tool.

## Deploy commands scripts can read

**`deploy list --json`.** CLI `0.4.39` (Oct 26) adds `--json` to `langgraph deploy list`, leaves environment variables out of the output, and fetches every page instead of only the first 20 deployments ([PR 9395](https://github.com/langchain-ai/langgraph/pull/9395)).

**`deploy metrics`.** CLI `0.4.42` (Nov 6) adds `langgraph deploy metrics`, which returns compact JSON grouped by section, chart and series, with `--summary` and `--raw` modes ([PR 9479](https://github.com/langchain-ai/langgraph/pull/9479)).

**Thread priority, in beta.** SDK `0.4.9` and CLI `0.4.38` (Oct 23) add a `Priority` object and a `priority` section in `langgraph.json`, so a deployment's own code can set each thread's run priority from -5 to 5 ([PR 9392](https://github.com/langchain-ai/langgraph/pull/9392), [PR 9393](https://github.com/langchain-ai/langgraph/pull/9393)). SDK `0.4.12` removed `fairness_weight` from `PrioritySettings`.

The summary above comes from LangGraph's release notes and the linked pull requests and was not independently tested.

Evidence:

- [LangGraph 1.2.24 release notes](https://github.com/langchain-ai/langgraph/releases/tag/1.2.24) (2026-11-20)
- [LangGraph 1.2.23 release notes](https://github.com/langchain-ai/langgraph/releases/tag/1.2.23) (2026-11-19)
- [prebuilt 1.1.3 release notes](https://github.com/langchain-ai/langgraph/releases/tag/prebuilt%3D%3D1.1.3) (2026-11-19)
- [checkpoint-postgres 3.2.1 release notes](https://github.com/langchain-ai/langgraph/releases/tag/checkpointpostgres%3D%3D3.2.1) (2026-11-14)
- [CLI 0.4.42 release notes](https://github.com/langchain-ai/langgraph/releases/tag/cli%3D%3D0.4.42) (2026-11-06)
- [SDK 0.4.14 release notes](https://github.com/langchain-ai/langgraph/releases/tag/sdk%3D%3D0.4.14) (2026-11-06)
- [LangGraph 1.2.20 release notes](https://github.com/langchain-ai/langgraph/releases/tag/1.2.20) (2026-11-05)
- [SDK 0.4.13 release notes](https://github.com/langchain-ai/langgraph/releases/tag/sdk%3D%3D0.4.13) (2026-11-05)
- [SDK 0.4.12 release notes](https://github.com/langchain-ai/langgraph/releases/tag/sdk%3D%3D0.4.12) (2026-11-03)
- [SDK 0.4.11 release notes](https://github.com/langchain-ai/langgraph/releases/tag/sdk%3D%3D0.4.11) (2026-10-30)
- [LangGraph 1.2.18 release notes](https://github.com/langchain-ai/langgraph/releases/tag/1.2.18) (2026-10-26)
- [CLI 0.4.39 release notes](https://github.com/langchain-ai/langgraph/releases/tag/cli%3D%3D0.4.39) (2026-10-26)
- [SDK 0.4.9 release notes](https://github.com/langchain-ai/langgraph/releases/tag/sdk%3D%3D0.4.9) and [CLI 0.4.38 release notes](https://github.com/langchain-ai/langgraph/releases/tag/cli%3D%3D0.4.38) (2026-10-23)
- [LangGraph GitHub repository](https://github.com/langchain-ai/langgraph)
- [LangGraph on Open Orchestrators](https://openorchestrators.org/players/langgraph/)
