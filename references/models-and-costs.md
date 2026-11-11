# Models, costs, and the evidence behind the routes

Figures are Claude API list prices and Anthropic's published results as of November 2026 (source: "Which Claude model should you build with?", claude.dev). Partner clouds price separately.

## Contents

- Side by side
- Cost per finished task
- What each effort level buys
- Latency
- Context cost
- Haiku's two price tiers
- Multi-agent results

## Side by side

| | Haiku 5.5 | Sonnet 5.5 | Opus 5.5 | Fable 5.2 |
|---|---|---|---|---|
| Model ID | `claude-haiku-5-5` | `claude-sonnet-5-5` | `claude-opus-5-5` | `claude-fable-5-2` |
| Built for | High-volume, cost-sensitive tasks | Fast, capable help with well-scoped tasks | Daily driver for complex coding and knowledge work | The most ambitious, open-ended technical work |
| Relative latency | Fastest | Fast | Moderate | Moderate |
| Input / output per MTok | $0.10 / $0.50 up to 100K-token prompts; $0.50 / $2.50 over | $2 / $10 | $4 / $20 | $8 / $40 |
| Cache reads per MTok | $0.01 up to 100K; $0.05 over | $0.10 | $0.20 | $0.20 |
| Context window | 1M | 1M | 1M | 1M |
| Max output | 128K | 128K | 128K | 128K |
| Thinking on the API | Adaptive, on by default, can be turned off | Adaptive, on by default; `between_tools` skips upfront thinking | Adaptive, always on | Adaptive, always on |
| Default effort on the API | medium | high (medium in Claude Code) | medium | high (medium in Claude Code) |
| Fast mode | No | No | Yes | Yes |

In Claude Code, `/effort` shows the session's current effort level.

## Cost per finished task

A higher price per token does not always mean a higher cost per task, or a better result. Compare models on what a finished task costs.

- Terminal-Bench 3.0: Fable 5.1 at `max` scored 58.0%; Opus 5.5 at `high` scored 58.9% with half the tokens. The smaller model at a sensible effort beat the larger model at its ceiling.
- FrontierCode 2.0: Fable 5.2 at `high` scored 45.3%, level with Opus 5.5 at `max` (45.5%), for roughly half the cost per task. On the hardest work the price gap closes, so before raising Opus past `high` on a hard task, test Fable.
- Small models do well on high-volume work with checkable outputs and are less suited to long agentic loops.
- Price the hardest tasks in a set as well as the typical ones, because they carry most of the spend. On one 20-problem search benchmark, two problems carried 43% of the spend.

## What each effort level buys

Opus 5.5 on SWE-bench Pro, relative to `high`:

| Effort | Score | Cost |
|---|---|---|
| `low` | about 8 points lower | about a third |
| `medium` | about 2.5 points lower | about 70% |
| `high` | baseline | baseline |
| `xhigh` | about 1.4 points higher | 2.5 times |

Low with retries: on the same subset, Opus 5.5 at `low` with failures re-run at `high` passed about 97% of tasks for about $0.17 each, against 95.3% for $0.29 with every task at `high`. This needs an automatic outcome check.

Giving a model a way to verify its own work is often more cost-effective, and adds less latency, than raising effort.

## Latency

- A faster setup finishes sooner and can miss more. Decide how long a person can wait before picking model and effort. For replies a person waits on, start Sonnet 5.5 at `low` or `medium`.
- Effort and latency rise together.
- Newer generations are often faster: Fable 5.2 generates output up to 30% faster than Fable 5.1, and Sonnet 5.5 runs 30% faster than Sonnet 5.
- To get a faster reply from the same model, pay for speed: on the Claude API, Opus 5.5 and Fable 5.2 offer fast mode, with higher output speed at premium pricing.

## Context cost

Every current model has a 1M-token context window. Large context costs on every turn because each turn resends it, though on long runs most input bills as cache reads.

- Opus 5.5 and Fable 5.2 both charge $0.20 per MTok for a cache read; Sonnet 5.5 charges $0.10. On a long, cache-heavy run the Fable-Opus gap comes down to uncached input and output, where Fable costs twice as much per token.
- On Opus 5.5, one turn's cache read costs about $0.03 at 150K tokens of context and about $0.004 at 20K.
- A lean main context matters more the bigger the main model is. Keep the expensive model's context for decisions, and let cheaper subagents do the verbose reading.

## Haiku's two price tiers

Haiku 5.5 is the only model whose token price varies with prompt length. A prompt up to 100K tokens bills at the lower tier; a prompt over 100K bills at five times that rate, which is still a quarter of Sonnet 5.5's per-token price.

Haiku pays off most on short prompts, where it costs a twentieth of Sonnet's input price. When every call needs a long prompt, compare Haiku with Sonnet on cost per finished task, not list price.

## Multi-agent results

- Lead and workers: on an internal defect-search benchmark over about 22 million tokens of source code, an Opus 5.5 lead over Sonnet 5.5 workers (up to 25 at a time) cost about a third less than Opus 5.5 alone at its default effort and scored about the same.
- Advisor: an Opus 5.5 executor at `high` with a Fable 5.1 advisor scored 90.1% at $2.92 per attempt on an internal coding benchmark. That is 1.7 points over Opus 5.5 alone at `high` for about 2.1 times the money: a gain at the edge of run-to-run noise, and about what raising effort would have bought. Of the four pairings in Anthropic's advisor table, none clearly beat its executor alone.
