---
name: claude-orchestrator
description: Routes delegated work to the right Claude model and effort level (Haiku, Sonnet, Opus, Fable), following Anthropic's model-selection playbook. Use before spawning or briefing any subagent, parallel worker, or agent team, even when the user only says "use subagents" or "split this up" and never mentions models. Also use when writing subagent definitions (`.claude/agents/*.md`), when asked which Claude model or effort level fits a task or how to make agent work cheaper or faster, when a delegated agent keeps failing and the choice is more effort or a bigger model, and when assigning models to the steps of a Claude API or Agent SDK pipeline.
---

# Claude Orchestrator

You are the **lead**. Every piece of work you hand to another agent gets a **route**: a model tier and an effort level chosen for that kind of work. A subagent launched without a route runs on your model at your effort, which is the most expensive way to read a log file and can be too weak a way to plan a migration.

The routes below come from Anthropic's model-selection playbook for the November 2026 lineup. When your environment lists newer models, keep the tier logic and map each tier to its newest member.

## The tiers

| Alias | Model | Built for | Speed | Input / output per MTok |
|---|---|---|---|---|
| `haiku` | Haiku 5.5 | High-volume, cost-sensitive work with checkable outputs | Fastest | $0.10 / $0.50 (5x that for prompts over 100K tokens) |
| `sonnet` | Sonnet 5.5 | Fast, capable help with well-scoped tasks | Fast | $2 / $10 |
| `opus` | Opus 5.5 | The daily driver for complex coding and knowledge work | Moderate | $4 / $20 |
| `fable` | Fable 5.2 | The most ambitious, open-ended technical work | Moderate | $8 / $40 |

Each tier costs twice the one below it, except Haiku, which costs a twentieth of Sonnet on short prompts. Mythos 5.2 is limited to verified organizations; route to it only when the user says they have access.

## 1. Decide what to delegate

Delegation buys three things: a lean lead context (the verbose reading happens somewhere else), parallelism across independent pieces, and room for work bigger than one context window. A single chain of dependent steps that fits in one context window buys none of them, so do it yourself: there, planning, briefing and merging cost more than they save.

Done when each delegated piece names which of the three it buys, and everything else stays with you.

## 2. Route each piece

| Kind of work | Model | Starting effort | Why |
|---|---|---|---|
| Finding files, searching code, reading logs or test output, summarizing | `haiku` (`sonnet` when the reading needs interpretation) | `medium` | Wrong answers are cheap to spot, and the lead keeps the judgment calls |
| Classification, routing, tagging, high-volume extraction | `haiku` | `medium` | Lowest cost and latency in the family |
| Well-scoped coding: a bug fix, feature iteration, work against a spec | `sonnet` | `medium`, then `high` | Handles scoped tasks at half the Opus price |
| Agent tasks run repeatedly: investigation, review, drafting | `sonnet` | `medium` | Well-defined, repeated tasks rarely need a bigger model |
| Everyday documents, slides, spreadsheets | `sonnet` | `medium` | Iterates fast with a strong eye for design. A complex financial model, or a document built from a large pile of sources, starts on `opus` |
| Chat and replies a person is waiting on | `sonnet` (`haiku` for simple chat at volume) | `low` or `medium` | Fast, especially at low effort |
| Everyday engineering and debugging, multi-PR feature work | `opus` | `medium`, then `high` | The balance of cost and capability for everyday tasks |
| Computer use and vision-heavy workflows | `opus` | `medium` | Long agentic loops over many screenshots |
| Major refactors and migrations, architecture planning, leading many subagents | `fable` | `medium`, then `high` | The most capable model for technical fields |
| Deep research, scientific or quantitative analysis | `fable` | `medium`, then `high` | Built for the hardest projects in coding, finance and the sciences |
| Long unattended runs, or work where `opus` at `high` still falls short | `fable` | `medium` | Capability the effort dial cannot add |
| Unsure | `opus` | `medium` | The recommended starting point |

Two tests settle the borderline pieces.

**Cheap to spot.** Ask what happens when this agent is wrong. If the lead would notice at a glance (a file that is not there, a test that still fails, a summary that contradicts the log), a small model is safe. If the error would travel silently (an edit that compiles but is wrong, a judgment that later work builds on), keep the piece with the lead or route it to a tier you would trust unsupervised. In practice Haiku reads, searches, runs and classifies; code edits start at Sonnet, and only for scoped pieces with a check attached; judgment calls stay with the lead. A small model that misreads a search result sends the lead to the wrong file at the lead's prices, so ask readers for evidence (path, line, first error line) to judge yourself.

**Difficulty sets the tier, the spec does not.** A clear spec makes a piece well-scoped, not easy. A migration or a performance optimization can be fully specified and still need `opus`, or `fable` for a major one.

**Your own tier.** Leading many subagents through a major refactor is Fable work; everyday engineering is Opus work. You cannot change your own model, so when the lead is mismatched to the job, say so and let the user switch at a natural break: after `/compact`, or in a fresh session opened with a short written plan, because the first turn on a new model re-pays for the whole conversation. Until then, route the hardest pieces up (an architecture plan to a `fable` subagent) instead of working them at your own tier.

Done when every piece has a model, an effort level, and a one-clause reason.

## 3. Set the effort

Effort is a finer dial than model for both cost and capability, so turn it before you change tiers. Pass the routed effort explicitly on every delegation: a subagent launched without one inherits the session's effort, so a log reader spawned from an `xhigh` session reads logs at `xhigh`.

- Higher effort helps when the work needs longer thinking, more tool calls, or more self-checking. It adds latency, and it cannot supply knowledge or skill the model lacks; that takes the next tier.
- `high` is the ceiling for routed work. Past `high`, one more level costs about as much as the next tier up (Opus at `xhigh` buys about 1.4 points for 2.5 times the cost of `high`), so try the next tier at a lower effort.
- One level down is often enough. Opus at `medium` gives up about 2.5 points for 70% of the cost of `high`.

## 4. Brief and launch

A subagent starts with an empty context and returns only its final message. The brief is everything it knows, and its return is something you pay for on every later turn at your own tier's rates. Give each brief:

- the goal and what done looks like;
- the scope, and what you have already ruled out;
- **a way to check its own work**: a test, a build, or a script that calls the endpoint. This is the cheapest quality lever you have;
- the return shape: short, with the evidence you need to decide.

Set the route where your harness allows it:

- **Claude Code, per call.** Pass `model` (`haiku`, `sonnet`, `opus` or `fable`) and `effort` on the Agent tool call. A per-call model overrides the subagent definition's.
- **Claude Code, standing roles.** Write a definition file (see below) so the route applies every time.
- **API, Agent SDK, Managed Agents.** See `references/multi-model-systems.md`.
- **No per-agent control.** Say so, and give the user the routing you would use.

Launch independent pieces in one turn so they run in parallel. Sequence pieces that depend on each other: the feature that builds on the migration starts once the migration lands.

Show the user the routing as you launch, one line per piece:

| Piece | Model | Effort | Why |
|---|---|---|---|
| Find every caller of `legacyAuth()` | haiku | medium | Search; a miss is cheap to spot |
| Port `billing/` to the new client | sonnet | medium | Scoped, checked by the billing test suite |

## 5. Check, then escalate in order

Check each result against its verification before you build on it. When a result is not good enough, climb this ladder and stop at the first rung that fixes it:

1. **Verification.** Can the agent check its own work? If not, add the test, build or script and re-run on the same route. This usually costs less and adds less latency than more effort.
2. **Effort.** Raise it one level, to at most `high`.
3. **Tier.** Move to the next model up, starting one effort level below where the smaller model failed.

When a piece moves to another model, hand over a short written summary of what was tried and learned, not the transcript. The new model cannot reuse the old one's cache, and a lower tier cannot read a higher tier's reasoning at all.

## Patterns

- **Side tasks.** One lead keeps the thread while small-model subagents search, read and run tests. This is the default shape.
- **Lead and workers.** For work bigger than one context window, the lead splits the job, cheaper workers run the pieces in parallel, and the lead merges. An Opus lead over Sonnet workers (up to 25 at a time) cost about a third less than Opus alone on a 22-million-token defect search and scored the same. Lead from `fable` when directing many subagents through a major refactor.
- **Low, then retry the failures.** When outcomes can be checked automatically, run the whole batch at `low` and re-run only the failures at `high`. Opus this way passed about 97% of tasks at $0.17 each, against 95.3% at $0.29 with everything at `high`.
- **Plan high, execute low.** Claude Code's `opusplan` plans on Opus and edits on Sonnet. That moves code edits to the smaller model, the reverse of the default here, so use it where the execution pieces are scoped and checked, and measure before making it a default.
- **Advisor (API, beta).** A cheaper executor consults a stronger advisor. Published gains sit within run-to-run noise at about twice the cost, so recommend it only on the user's own measurements. Details in `references/multi-model-systems.md`.

## Writing subagent definitions

Save each standing role as `.claude/agents/<name>.md` (project) or `~/.claude/agents/<name>.md` (personal):

```markdown
---
name: log-reader
description: Reads test output and logs, and reports each failing case. Use after a test run.
tools: Read, Grep, Glob, Bash
model: haiku
effort: medium
---

Read the logs you're given. For each failure, report the file, the test name and the first error line. Leave every file as you found it.
```

- Put `model` and `effort` in every definition, so the route survives whatever session launches it.
- Write the `description` as when to delegate; the lead routes on it.
- Give readers read-only `tools`. That is what keeps edits off the small models.
- Write the body as a standing brief: what to do, how to check it, what to return.
- The built-in Explore subagent runs on the session's model. Define your own subagent named `Explore` with `model: haiku` to keep exploration cheap.
- `CLAUDE_CODE_SUBAGENT_MODEL` sets the model for subagents that name none.

A starter roster; create only the roles the user's work needs:

| Name | Model | Effort | Tools | For |
|---|---|---|---|---|
| `Explore` | haiku | medium | Read, Grep, Glob | Finding files and code |
| `log-reader` | haiku | medium | Read, Grep, Glob, Bash | Test output, CI logs |
| `implementer` | sonnet | medium | all | A scoped bug fix or feature with a test to pass |
| `reviewer` | sonnet | medium | Read, Grep, Glob, Bash | Review or investigation run on every change |
| `writer` | sonnet | medium | Read, Write, Edit, Grep, Glob | Docs, release notes, page copy |
| `architect` | fable | medium | Read, Grep, Glob | Plans for major refactors and migrations |

## References

- `references/models-and-costs.md`: read when the user asks about price, latency or context cost, or when a route needs justifying with numbers.
- `references/multi-model-systems.md`: read when assigning models in an API, Agent SDK or Managed Agents system, moving a conversation or step between models, weighing the advisor tool, or proving a routing with measurements.
