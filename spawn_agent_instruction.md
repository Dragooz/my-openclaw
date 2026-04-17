mkdir -p ~.openclaw/worksapce-{your-agent-motive}/skills
nano ~.openclaw/workspace-{your-agent-motive}/skills/{your-skill}.md - Put in your skill .md here
openclaw agents add {your-agent-name} - It will initialize an agent for you, and walk through the GUI; Make sure the workspace match;
nano ~/.openclaw/openclaw.json - Ensure that the agent setup in it, is correct; Note: I removed model settings in the agent, and rely only the master setup.
rm ~./openclaw/workspace-{your-agent-motive}/AGENTS.md && nano ~./openclaw/workspace-{your-agent-motive}/AGENTS.md - Replace with your intruction to the agent;
openclaw cron add \
 --name "{your-cron-name}" \
 --cron "0 8 \* \* \*" \
 --tz "Asia/Kuala_Lumpur" \
 --session isolated \
 --agent {your-agent-name} \
 --message "{Your cron Command}" \
 --announce \
 --channel telegram \
 --to "user:{your-telegram-bot-id}"
sed -i '' 's|https://openrouter.ai/v1|https://openrouter.ai/api/v1|g' \
 ~/.openclaw/agents/{your-agent-name}/agent/models.json - This is to fix baseURL bug.
