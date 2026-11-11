# Building and changing multi-model systems

Source: "Which Claude model should you build with?" (claude.dev, November 2026) and the Claude Code subagent docs.

## Contents

- Setting the route on each surface
- Moving work between models
- Compatibility checks before a step changes model
- One request shape for every model
- Advisor tool
- Measure on your own work

## Setting the route on each surface

**Claude Code.** A subagent's model resolves in this order:

1. the `model` passed on the Agent tool call (aliases only: `haiku`, `sonnet`, `opus`, `fable`);
2. the `model` in the subagent definition's frontmatter (an alias, a full model ID, or `inherit`);
3. the `CLAUDE_CODE_SUBAGENT_MODEL` environment variable;
4. the main conversation's model.

Effort comes from the `effort` on the call or in the definition (`low`, `medium`, `high`, `xhigh`, `max`), and otherwise from the session.

- The built-in Explore subagent runs on the session's model (capped at Opus on the Claude API) instead of always running on Haiku. Define a subagent named `Explore` with `model: haiku` to keep exploration cheap.
- On the Anthropic API the `haiku` alias resolves to Haiku 5.5, which needs Claude Code v2.1.293 or later. On Amazon Bedrock, Google Cloud, Microsoft Foundry and Claude Platform on AWS it still resolves to Haiku 4.5. Pin what the alias means with `ANTHROPIC_DEFAULT_HAIKU_MODEL`.
- The `opusplan` alias plans on Opus in plan mode and executes on Sonnet, which puts code edits on Sonnet. Measure it on the user's tasks before making it a default.

**Claude API and Agent SDK.** Name the model per request with its full ID (`claude-haiku-5-5`, `claude-sonnet-5-5`, `claude-opus-5-5`, `claude-fable-5-2`) and set effort explicitly, because API defaults differ from Claude Code's: Sonnet 5.5 and Fable 5.2 default to `high` on the API, Opus 5.5 and Haiku 5.5 to `medium`.

When the routing rule is simple, such as one model for each step type, let code pick the model instead of asking a model to.

**Claude Managed Agents.** Multiagent orchestration gives a coordinator agent and a roster of workers. Use it for work bigger than one context window. Skip it for one chain of dependent steps or work that fits in one context window, where planning and merging cost more than they save.

## Moving work between models

**The cache belongs to the model that wrote it.** The first turn on a new model pays the cache-write price on the whole conversation. Make that turn small:

- in Claude Code, run `/compact` first, or start a fresh session with a short written plan;
- on the API, open the new model's conversation with a summary of the work so far instead of the full history.

**Thinking blocks record the model that produced them, and each model reads only some other models' blocks.**

| Model | Reads thinking blocks from |
|---|---|
| Fable 5.2 | Opus 5.5, Sonnet 5.5, Haiku 5.5 |
| Opus 5.5 | Sonnet 5.5, Haiku 5.5 |
| Sonnet 5.5 | Haiku 5.5 |
| Haiku 5.5 | none of the larger models |

This holds on the Claude API and Claude Models on Google Cloud (Google managed). On the Claude API, Fable 5.1 also reads Opus 5.5's blocks.

- A move down a tier always loses the earlier reasoning, because a model never reads the blocks of a model above it. A move from Fable 5.2 to any other generally available model drops its reasoning.
- When the new model cannot read the earlier blocks, the API drops them and the request still succeeds. The new model then works without that reasoning, silently.

So switch at a natural break, and carry the reasoning across in writing.

## Compatibility checks before a step changes model

Every current model supports tool use and image input. Test these differences before moving a request:

- **Preserved thinking.** Fable 5.2 enforces it on every API account: the API returns a 400 when anything before a Fable 5.2 thinking block has changed (the system prompt, the tools, or an earlier message), unless the request opts in to dropping the affected thinking. Requests built through Claude Code, the Agent SDK or Managed Agents need no change. Code that edits history must become append-only.
- **Thinking.** Opus 5.5 and Fable 5.2 always think; a request that turns thinking off returns an error. Sonnet 5.5 thinks by default, and `between_tools` limits thinking to the gaps between tool calls. Haiku 5.5 thinks by default and allows thinking off at `high` effort or below. On all of them, read content blocks by type, because a response can begin with a thinking block.
- **Forced tool use.** A `tool_choice` of `any` or `tool` returns a 400 on Sonnet 5.5, Opus 5.5 and Fable 5.2. Send `auto`, mark the tool `strict: true`, and say in the prompt when to use it.
- **Computer use.** All four models support it only through the `computer_toolset_20260801` toolset on the Claude API and Claude Models on Google Cloud (Google managed).
- **Haiku 4.5 to Haiku 5.5.** Replace `budget_tokens` with an effort level. Drop non-default `temperature`, `top_p` and `top_k` values, which return a 400. End messages with a user turn; a final assistant turn for the model to continue also returns a 400.
- **Safety fallbacks.** Opus 5.5, Sonnet 5.5 and Fable 5.2 run safety classifiers that most often flag cybersecurity and biology content. In Claude Code a flagged request can re-run on an older model, with a notice in the transcript naming it. Fable 5.2 falls back to Opus 4.8 for flagged cybersecurity requests and Opus 5 for flagged biology requests.

## One request shape for every model

When one system runs several models, build every request for the strictest of them:

- keep history append-only;
- send `tool_choice: auto` with strict tools;
- read content blocks by type;
- write prompts and tool descriptions precisely enough for the most literal model.

Then a step can move up or down a tier without a code change.

## Advisor tool

The advisor tool (beta) pairs a faster, cheaper executor with a stronger advisor. The executor runs the task and can ask the advisor for guidance partway through. Most tokens bill at the executor's rate.

- It helps only when the advisor adds capability the executor lacks and the executor asks, which it does less often at low effort. Measure the executor's consult rate first.
- It might pay off on agentic work where most turns are routine.
- Anthropic's own cost guide is cautious: none of its four published pairings clearly beat the executor alone (numbers in `models-and-costs.md`).
- Check which advisors the executor accepts. A Sonnet 5.5 executor accepts Fable 5.2, Opus 5.5, Opus 5 and Sonnet 5.5. A Fable 5.2 executor accepts only Fable 5.2 or Mythos 5.2.

## Measure on your own work

Published benchmarks pick a starting model. Before committing to a routing:

1. Pull a sample of real tasks. Write an outcome check for each: a passing test, a closed ticket, a correct row count.
2. Run each candidate model at two or three effort levels, each in its own session so the runs do not share a cache. Record the cost of each completed task from the `usage` field of each response.
3. Plot score against spend. Keep a multi-model setup only if it beats every single-model point on that chart.
4. Run the winner in shadow on a slice of real traffic before cutting over.

For a quick check in Claude Code, run one backlog task on two models with `/model` and compare what `/usage` reports.
