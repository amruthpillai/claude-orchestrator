---
tags: [skill-creator]
max_turns: 20
timeout_seconds: 600
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash]
---

ok I've got a pile of work on our storefront repo (~600 files, Express + plain JS backend, React front) and I want it split across subagents rather than one giant session:

1. migrate the whole backend from JavaScript to TypeScript
2. build monthly subscriptions: new tables, Stripe webhooks, checkout changes, an account page. probably 5 or 6 PRs. it has to be written in TS on top of the migrated code
3. rewrite the ~40 product page descriptions in content/products/ (marketing sent a tone guide)
4. tag the ~12,000 support emails in data/inbox.jsonl as billing / shipping / returns / other
5. CI has been flaky for two weeks and nobody knows why. there's ~300 MB of logs in ci-logs/

Don't start any of it yet. Write the delegation plan to delegation-plan.md: which agent does what, on which model, and how you'd run the whole thing.
