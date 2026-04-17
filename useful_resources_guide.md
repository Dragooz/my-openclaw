Official Docs (docs.openclaw.ai)

https://docs.openclaw.ai/cli/agents — agents CLI reference (the one that corrected the --identity mistake)
https://docs.openclaw.ai/cli/cron.md — cron CLI reference (found edit <job-id> positional syntax)
https://docs.openclaw.ai/concepts/agent-workspace — workspace structure, standard files
https://docs.openclaw.ai/automation/cron-jobs — cron jobs full guide (delivery flags, --announce, --channel, --to)

GitHub

https://github.com/openclaw/openclaw — main repo (confirmed .agents folder, 348k stars)
https://github.com/openclaw/openclaw/blob/main/AGENTS.md — internal agent coding rules
https://github.com/openclaw/openclaw/issues/43177 — bug: cron announce reports delivered without actual Telegram delivery
https://github.com/openclaw/openclaw/issues/18573 — bug: cron announce routes to @heartbeat instead of chat ID
https://github.com/openclaw/openclaw/issues/13812 — bug: v2026.2.9 cron announce sends summary instead of full output
https://github.com/openclaw/openclaw/issues/14743 — bug: cron announce delivery to Telegram silently fails
https://github.com/shenhao-stu/openclaw-agents — community multi-agent setup kit (confirmed agents add <id> syntax)

Third-party guides

https://www.meta-intelligence.tech/en/insight-openclaw-agents-guide — agents add/list/config reference
https://www.crewclaw.com/blog/openclaw-cli-commands-reference — CLI cheat sheet
https://lumadock.com/tutorials/openclaw-skills-guide — skills/SKILL.md format
https://lumadock.com/tutorials/openclaw-cli-config-reference — full config reference, openclaw.json
https://lumadock.com/tutorials/openclaw-cron-scheduler-guide — cron scheduler patterns
https://www.stack-junkie.com/blog/openclaw-cli-commands-reference — operator CLI flags
https://www.stack-junkie.com/blog/openclaw-cron-jobs-automation-guide — cron job setup
https://pub.towardsai.net/openclaw-complete-guide-setup-tutorial-2026-14dd1ae6d1c2 — full setup tutorial, workspace file explanation
https://amankhan1.substack.com/p/how-to-make-your-openclaw-agent-useful — SOUL.md, USER.md, AGENTS.md practical guide
https://moltfounders.com/openclaw-configuration — openclaw.json full config reference
https://moltfounders.com/openclaw-mega-cheatsheet — 150+ CLI commands cheatsheet
https://openclaw-ai.com/en/docs/automation/cron-jobs — mirror of cron jobs docs
https://skywork.ai/skypage/en/ultimate-openclaw-cli-commands/... — CLI commands guide
https://generect.com/blog/openclaw-ai-agent/ — setup walkthrough
https://www.kdnuggets.com/openclaw-explained-... — general overview
https://www.digitalocean.com/resources/articles/what-is-openclaw — skills system overview
https://milvus.io/blog/openclaw-formerly-clawdbot-moltbot-explained... — architecture deep dive
https://en.wikipedia.org/wiki/OpenClaw — history, general context
https://lobehub.com/skills/oabdelmaksoud-openclaw-skills-openclaw-workspace-structure — workspace structure skill
https://www.learnclawdbot.org/docs/tools/skills — skills system docs mirror
https://open-claw.bot/docs/tools/skills/ — another skills docs mirror
https://near.ai/openclaw — NEAR AI hosted OpenClaw (TEE deployment context)

For next time, the most authoritative sources in priority order are:

https://docs.openclaw.ai/cli/* — ground truth for exact CLI syntax
https://docs.openclaw.ai/automation/cron-jobs — for scheduling
https://docs.openclaw.ai/concepts/agent-workspace — for workspace file structure
https://docs.openclaw.ai/llms.txt — index of all doc pages (useful to give me upfront)
GitHub issues — real-world bugs and workarounds
