---
tags: [skill-creator]
max_turns: 20
timeout_seconds: 600
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash]
---

can you set up subagents for our team in .claude/agents/? we're a 6-person team on a Django monorepo and the Claude bill is getting silly because everything runs on whatever model the session is on. I want: one for finding stuff in the codebase, one that reads pytest output / CI logs after a run and tells us what failed, one that fixes small well-specified bugs from Linear tickets, one that reviews PR diffs (we run it on every PR), and one for planning the big cross-service refactors we do maybe once a quarter. pick the right model for each.
