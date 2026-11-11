# Claude Orchestrator

An agent skill that makes Claude pick the right model and effort level for every piece of work it delegates.

Without it, a subagent runs on whatever model and effort the session happens to be on. That is the most expensive way to read a log file, and sometimes too weak a way to plan a migration. With it, Claude acts as the lead and gives each delegated piece a **route**: a model tier (Haiku, Sonnet, Opus or Fable) and an effort level chosen for that kind of work.

## Where it comes from

The skill packages the guidance in Anthropic's playbook **[Which Claude model should you build with? Fable, Opus, Sonnet, or Haiku](https://claude.dev/blog/which-claude-model-should-you-build-with/)** by Addy Osmani (November 2026). The routing table, the effort rules, the escalation order and every number in the reference files come from that article. Read it for the full reasoning and the demos; the skill is the part of it an agent can act on.

## What the skill covers

- **A routing table** from kind of work to model and starting effort.
- **Effort rules.** Set effort on every delegation, treat `high` as the ceiling, and try the next model at a lower effort instead of going past `high`.
- **An escalation order** for when a delegated agent falls short: give it a way to verify its own work, then raise effort one level, then move up one model.
- **Delegation patterns.** Side-task subagents, a lead with cheaper parallel workers, running a batch at low effort and retrying only the failures, and cautions on `opusplan` and the advisor tool.
- **Subagent definitions.** A template and starter roster for `.claude/agents/*.md` with `model` and `effort` set, including a Haiku `Explore` that replaces the built-in one.
- **Two reference files**, loaded only when needed: prices and benchmark evidence, and the rules for building or changing a multi-model system on the Claude API.

The routing at a glance:

| Kind of work | Model | Starting effort |
|---|---|---|
| Finding files, reading logs or test output, summarizing | Haiku (Sonnet when it needs interpretation) | medium |
| Classification, tagging, high-volume extraction | Haiku | medium |
| Well-scoped coding: a bug fix, work against a spec | Sonnet | medium, then high |
| Repeated agent tasks: review, investigation, drafting | Sonnet | medium |
| Everyday documents, slides, spreadsheets | Sonnet | medium |
| Everyday engineering, debugging, multi-PR features | Opus | medium, then high |
| Computer use and vision-heavy workflows | Opus | medium |
| Major refactors and migrations, architecture, leading many subagents | Fable | medium, then high |
| Deep research, scientific or quantitative analysis | Fable | medium, then high |
| Unsure | Opus | medium |

The full table, with the reasoning for each row, is in [`SKILL.md`](SKILL.md).

## Install

For Claude Code, clone the repository into your skills folder:

```bash
# personal: available in every project
git clone https://github.com/amruthpillai/claude-orchestrator ~/.claude/skills/claude-orchestrator

# or per project
git clone https://github.com/amruthpillai/claude-orchestrator .claude/skills/claude-orchestrator
```

Start a new session and the skill appears as `claude-orchestrator`. Other agents that read the `SKILL.md` format can load the same folder.

## When to use it

Claude loads the skill on its own in these situations, and you can also call it directly with `/claude-orchestrator`:

- You ask Claude to use subagents, split work up, or run things in parallel.
- You ask for subagent definitions in `.claude/agents/`.
- You ask which Claude model or effort level fits a task, or how to make agent work cheaper or faster.
- A delegated agent keeps failing and you are deciding between more effort and a bigger model.
- You are assigning models to the steps of a Claude API or Agent SDK pipeline.

It is not needed for a single chain of dependent steps that fits in one context window. The skill itself tells Claude to do that work directly, because planning and merging would cost more than they save.

## Example

Say you give Claude this:

> I've got a pile of work on our storefront repo and I want it split across subagents: migrate the backend from JavaScript to TypeScript, build monthly subscriptions on top of the migrated code, rewrite the 40 product page descriptions, tag 12,000 support emails by category, and work out why CI has been flaky from 300 MB of logs. Write me the delegation plan.

With the skill, the plan routes each piece instead of running everything on the session's model:

| Piece | Model | Effort | Why |
|---|---|---|---|
| Plan the migration, convert the core modules, close it out | Fable | medium, then high | A major migration; every later batch builds on it |
| Convert the remaining directories in parallel batches | Sonnet | medium, then high | Scoped by the plan and checked by typecheck and tests |
| Build subscriptions, one agent per PR, after the migration lands | Opus | medium, then high | Multi-PR feature work on a money path |
| Rewrite the product descriptions | Sonnet | medium | Everyday copy against a tone guide |
| Tag the support emails | Haiku | medium | High-volume classification with checkable output |
| Extract failures from the CI logs | Haiku | medium | Log reading; a wrong extract is cheap to spot |
| Diagnose the flakiness from those extracts | Opus | medium, then high | The judgment call of the job |

Every agent also gets a check it can run itself, nothing starts above `high`, and the plan says what happens when a result falls short.

A second example, for the escalation order. You ask:

> My Sonnet subagent has failed twice at getting our /reports endpoint under 800ms p95. It's already at high effort and I check the number by hand in Grafana. Should I rerun it at max?

With the skill, the answer is no, followed by an order of steps: first give the agent a benchmark script so it can measure p95 itself and rerun once on the same settings; if that misses, move to Opus at medium effort with a short written summary of what was tried; only then raise Opus to high.

## Test results

The repository includes the three test cases in [`evals/`](evals), in the layout `claude plugin eval` reads. Each was run once with the skill and once without, on Claude Fable 5.2:

| Test | With skill | Without skill |
|---|---|---|
| Delegation plan for a five-part workload | 9 of 9 | 6 of 9 |
| Five subagent definitions for a team | 8 of 8 | 6 of 8 |
| Escalation advice for a failing subagent | 6 of 6 | 2 of 6 |

The skill cost about 12% more tokens and 15% more time. These are single runs, so there is no variance estimate. The gain is largest on escalation advice and smallest on subagent definitions, where the model already picks sensible models unaided and the skill mainly adds the `effort` field and the `Explore` override.

## Layout

```
SKILL.md                            the routing steps, patterns and definition template
references/models-and-costs.md      prices, latency, context cost, benchmark evidence
references/multi-model-systems.md   API compatibility, switching models, advisor tool, measuring
evals/                              three test cases with their graders
```

## Keeping it current

Model names, prices and benchmark figures are those of the November 2026 lineup: Haiku 5.5, Sonnet 5.5, Opus 5.5 and Fable 5.2. The skill tells Claude to keep the tier logic and map each tier to its newest member when newer models appear, but the numbers in the reference files will age. Check them against the [original article](https://claude.dev/blog/which-claude-model-should-you-build-with/) and Anthropic's current pricing before relying on them, and measure any routing on your own tasks before committing to it.

## Credit

All of the guidance belongs to Anthropic and the article's author. This repository is an independent packaging of that article as a skill and is not affiliated with or endorsed by Anthropic.
