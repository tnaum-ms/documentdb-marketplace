---
name: documentdb-explain-plan
description: Understand or optimize a pasted DocumentDB or MongoDB API explain plan using a deterministic summary and a side panel. Use for explicit explain-plan help, not general database questions, live database work or query writing without a plan.
---

# DocumentDB Query Insights

1. Inspect capabilities for canvas ID `documentdb-explain-plan`.
2. Open it with a stable instance ID such as `documentdb-explain-plan-main` and
   input `{}` **before** invoking an action: actions need that instance ID.
   Reuse the instance when reopening; a known `planId` can attach another panel.
3. Call `load_plan` on that instance with `{ planText, label? }`. Pass the explain
   output unchanged, without Markdown fences (JSON, Extended JSON or shell BSON;
   at most 512 KB of UTF-8 text). If chat says the plan is too long or tool limits
   prevent passing it, open the canvas with `{}` and direct the user in plain text
   to paste the unchanged explain output in the panel's paste box (at most 512 KB),
   then end the turn. Do not use a blocking structured question awaiting a panel
   response. Never truncate the plan.
4. Initially answer briefly from the returned summary: rating if available, dominant stage,
   and one or two findings. Mention the side panel; do not dump the raw plan.
   Planner-only output has no measured timings.
5. Ask once per plan after the brief answer, including a fresh chat whose first
   message contains the request and plan together:
   "Can you share the {missing inputs}? They would sharpen this analysis."
   Replace `{missing inputs}` using the returned `missingInputs`: `indexDefinitions`
   means index definitions, `indexUsage` means index usage, `collectionStats` means
   collection stats, and `executionStats` means executionStats output.
   Use the host's structured question tool (for example `ask_user`) when available;
   use its host-specific contract without prescribing argument keys. Otherwise
   show the exact question and a short list in chat. Choices, in this order:
   - I'll share them
   - Tell me how to get these
   - Just explain this plan

   Never claim buttons or a form were shown without calling the tool.
   No commands in the initial question.
   On "I'll share them", reply "Paste them here in chat when ready."
   Index definitions, index usage and collection stats belong in chat only; the
   panel accepts explain output only. Do not offer the panel for these answers or
   ask where to send them.
   On "Tell me how to get these" or another explicit help request, give the needed
   commands only when asked, one per line with its purpose. Use the known
   collection name, otherwise `<coll>`:
   - `db.<coll>.getIndexes()` — index definitions.
   - `db.<coll>.aggregate([{ $indexStats: {} }])` — index usage.
   - `db.runCommand({ collStats: '<coll>' })` — collection stats.
   If executionStats is missing, show `.explain('executionStats')` on the same query.
   Never execute these commands.
   On "Just explain this plan", continue with the supplied facts and do not ask
   again for that plan, including improve-plan button turns.
6. The panel's explain-plan button requests thorough teaching of the whole plan:
   execution order, stages, terms defined once, rating and efficiency, with honest
   unknowns. Do not push recommendations or ask another question.
   The improve-plan button requests the full optimization method, not a brief
   summary: relevant scenarios, ranked stage-linked recommendations, expected
   effects, trade-offs, verification commands (never execute them), and conclusions
   limited by missing input. Preserve the once-per-plan question guard.
   For both whole-plan buttons, use the host's skill-loading tool to load the installed
   skill named "explain-plan" from the "documentdb-explain-plan" plugin. Resolve its
   actual qualified registered identifier if the host prefixes it; do not guess a tool signature.
   Confirm an actual successful load, then read the attached unchanged SKILL.md and matching
   scenario files before analyzing. If the skill-loading tool is unavailable, the skill is
   not found or disabled, loading fails, or required knowledge cannot be read, respond only
   with an actionable error explaining what failed, why, and how to enable or update the
   plugin and start a new chat; then STOP. Do not invent or generate a generic query
   description, plan analysis or recommendations as a fallback. Do not claim the skill
   loaded without an actual successful load. Loading file attachments is not proof that
   the skill-loading tool was invoked. Both whole-plan modes attach the existing unchanged
   skill files; missing or unreadable required files prevent the request from being sent.
   Initial deterministic previews and brief stage explanations do not require this skill.
   Native MongoDB API plans need native-engine interpretation, not DocumentDB-specific
   assumptions.
   Panel button requests attach the original explain output unchanged
   (`explain-output.json`, or `.txt` for shell syntax) as the only source of truth, and
   `panel-view.json`, which describes what the panel shows (labels, metric cards, time
   bars, panel rating and findings). Use the panel view only to point at what the user
   sees; when it disagrees with the original, trust the original.
7. Each whole-plan button request carries a fixed Markdown response format that matches the
   DocumentDB extension's Query Insights tab. Explain uses "Understanding Your Query Execution
   Plan" only; improve uses "Query Performance Analysis" and "Optimization Opportunities" only.
   Follow the format in the request exactly; it replaces the explain-plan skill's own output format.

Do not open for text PostgreSQL EXPLAIN, unrelated questions, live database work,
index creation, or query writing without an explain plan. This canvas never
connects to a database, executes commands or holds credentials. Keep advice
grounded in supplied facts; give collection commands only when asked, never run them here.
