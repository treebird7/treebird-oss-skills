---
name: ux-foolproof-review
description: Review any onboarding/setup/connect flow by executing it as a complete stranger — cold environment, zero credentials, zero tribal knowledge — and logging every point of friction as a product bug. Use when asked to "foolproof this flow", "test as a new user", "would a stranger survive this", or before publishing any install/connect/onboarding path.
---

# /ux-foolproof-review

Test a flow the way a **complete ignorant** meets it: no repo checkout, no env vars, no secrets manager, no memory of the design discussions. The reviewer's job is to *be* the stranger, not to imagine one.

## The prime rule

**Your warm shell is a liar.** Your machine has the creds, the PATH, the config files, and your head has the context. Every test must run in a genuinely cold context or it proves nothing:

- Shell flows: `env -i HOME=<empty-dir> PATH=/usr/bin:/bin <cmd>` from an empty scratch dir.
- Agent flows: spawn a subagent whose prompt contains ONLY the public artifact ("You have this join link and nothing else. Get into the chat.") — no repo paths, no lore.
- Web flows: incognito, logged out.
- If you can't fake cold (a machine-wide config, an already-logged-in CLI, or a cached credential leaks in), say so and name exactly what leaked.

## Method

1. **Inventory the stranger's hands.** Write down literally everything they start with (a URL, an install command, a join link). Everything else is an assumption to be caught.
2. **Walk the advertised path verbatim.** Follow the README/connect page/skill exactly as written — no improvising around gaps. Where you're forced to improvise, that's a finding.
3. **Log every friction point** as it happens, with the exact error text. Classify each:
   - `hidden-credential` — step silently requires a key/login the stranger lacks (worst class; a *service* key requirement is a security bug too)
   - `wrong-surface` — docs route the user to an endpoint/tool meant for a different client class
   - `baked-identity` — state fixed at install/start time that the user expects to change at use time
   - `path-assumption` — binary/file assumed present ("already in PATH")
   - `stale-doc` — instruction references removed tools/hosts/flags
   - `missing-hint` — flow works but the next step is only discoverable via tribal knowledge (fix: put the hint in the tool response/UI itself, once, not on every call)
4. **Fix at the root when you own the product.** Prefer making the cold path *work* (guest fallback, sensible default, hint in the response) over documenting around the failure. Docs-only fixes are for genuinely gated steps.
5. **Re-run cold after every fix.** A fix verified warm is unverified. Same `env -i` / fresh-subagent discipline.
6. **End-to-end finale:** one full run of the entire flow, cold, no interventions. It must succeed touching only what the stranger has.

## Deliverable

A friction log ordered by **where the stranger falls off first** (not by severity class): step → exact failure → class → root fix vs doc fix → verified-cold ✅/❌. Plus the one-paragraph answer: *can a stranger complete this today, yes or no, and what's the single biggest cliff.*

## Gotchas that cost real sessions

- Testing only the files you changed — run the whole suite/flow; adjacent steps inherit your assumptions.
- Sequencing: pipelined tool calls can race (identity declared in call N not yet persisted for call N+1) — test the way a real user paces, sequentially.
- "Restart to apply" — a stale running process shows pre-fix behavior; restart before declaring the fix dead.
- The stranger on a *different platform* (Codex vs Claude Code, Windows vs mac) hits different walls — enumerate client classes and cold-test each advertised one.
