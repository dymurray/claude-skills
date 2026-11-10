# Claude Code skills & commands

Personal Claude Code skills and slash commands for MTA/JIRA release work.

- `commands/` — slash commands (`/jira-issue-draft`, `/create-art-ticket`, `/create-spmm-ticket`): draft and create standards-compliant Jira issues in the MTA project (HPBU hierarchy, templates), ART tickets, and SPMM tickets.
- `skills/bump-zstream-version/` — bump z-stream versions across product repos.
- `skills/trigger-art-release/` — trigger an ART release via Konflux/Tekton pipelines.

Install by copying into `~/.claude/commands/` and `~/.claude/skills/` respectively.
Credentials come from environment variables (`JIRA_EMAIL`, `JIRA_API_TOKEN`); nothing is embedded.
