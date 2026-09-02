# Supatest AI - Shared Claude Skills

This repository contains organization-wide Claude skills for the Supatest AI team. These skills work across Claude Code, Claude.ai, and Claude Desktop.

## 📦 Installation

### Quick Install (Recommended)

```bash
# Clone and run the installer
git clone git@github.com:supatest-ai/skills.git /tmp/supatest-skills
cd /tmp/supatest-skills
./scripts/install.sh
```

The installer will:
- Install skills to `~/.claude/skills/supatest`
- Optionally set up automatic updates every 30 minutes
- Show you all available skills

### Install with Public Skills

To install company skills + recommended public skills:

```bash
cd /tmp/supatest-skills
./scripts/install-with-public.sh
```

This installs:
- Your company skills (supatest)
- Public skills from `public-skills.json` (e.g., Anthropic official skills)

All skills will be available across all your projects!

### Manual Installation

#### For Global Access (All Projects)

```bash
# Clone into your global skills directory
cd ~/.claude/skills/
git clone git@github.com:supatest-ai/skills.git supatest
```

#### For Project-Specific Use

```bash
# Clone into your project
cd your-project/.claude/skills/
git clone git@github.com:supatest-ai/skills.git supatest
```

## 🎯 Available Skills

### work-summary
Generate comprehensive work summaries from git commits across multiple repositories with task categorization.

**Usage:** `/work-summary <author> "<start_datetime>" "<end_datetime>" <repo1> [repo2]...`

**Example:** `/work-summary Prasad "2026-02-13 09:00" "2026-02-13 18:00" supatest`

### update-skills
Manually update Supatest skills to get the latest changes immediately.

**Usage:** Invoke with `/update-skills` to pull latest skills updates.

### signoz-health-check
Perform comprehensive health checks of the SigNoz observability platform, analyzing services, logs, metrics, traces, and alerts.

**Usage:** Invoke with `/signoz-health-check [timeRange]` to check system health (e.g., `/signoz-health-check 24h`).

### debug-prod
Investigate a production incident using a mitigation-first protocol. Uses the observability funnel (metrics → traces → logs), hypothesis-driven instrumentation, and produces a structured findings report and optional postmortem.

**Usage:** `/debug-prod [description of the issue]`

### bug
Structured bug investigation: capture → reproduce → root cause → write failing test → fix → verify. Supports regression detection via `git bisect` and optional GitHub issue filing.

**Usage:** `/bug [description of the bug]`

### review-pr
Review a pull request. Filters to issues introduced by the PR (not pre-existing), scores confidence, produces a prioritized severity report, and requires your approval before posting anything to GitHub.

**Usage:** `/review-pr [PR number or branch name]`

### deploy
Deploy your application to staging or production via GitHub Actions. Auto-detects changed services by diffing against the last successful deployment, shows a plan, and streams live workflow progress.

**Usage:** `/deploy [staging|prod] [services]`

**Example:** `/deploy prod web,api`

### commit
Commit staged files with well-structured commit messages following conventional commits format.

**Usage:** Invoke with `/commit` to commit already staged files.

### commit-all
Stage all changes and commit with well-structured commit messages following conventional commits format.

**Usage:** Invoke with `/commit-all` to stage and commit everything.

### mobile-test
Test Android and iOS apps end to end across critical flows, permissions, lifecycle transitions, network conditions, accessibility, logs, and visual evidence.

**Usage:** `/mobile-test [app-or-feature, build-path-or-repo, and critical-flows]`

**Example:** `/mobile-test "React Native offline sync — test Android and iOS; create a note offline, reconnect, and verify one synced note"`

### test-feature
Interactively test a completed feature using browser automation. Navigates the live app, exercises all key flows, captures a GIF walkthrough, and produces a visual verification report.

**Usage:** `/test-feature [feature-description]`

**Example:** `/test-feature "user authentication login form — app at http://localhost:3000"`

## 🔄 Auto-Update

Skills automatically update every 30 minutes in the background. The installer sets this up for you!

If you skipped auto-update during installation, enable it anytime:
```bash
cd ~/.claude/skills/supatest
./scripts/setup-launchd.sh
```

**Management commands:**
```bash
# View update log
tail -f ~/.claude/skills/supatest-update.log

# Check if updater is running
launchctl list | grep supatest

# Disable auto-updates
launchctl unload ~/Library/LaunchAgents/com.supatest.skills-updater.plist

# Re-enable auto-updates
launchctl load ~/Library/LaunchAgents/com.supatest.skills-updater.plist

# Manual update
cd ~/.claude/skills/supatest && git pull
```

## 🌟 Recommended Public Skills

These are curated public skills we recommend for the team. Use `/update-skills` to see which ones you have installed and get prompted to install missing ones.

### Official & Essential

| Skill Repository | Description | Install Command |
|-----------------|-------------|-----------------|
| [Anthropic Official Skills](https://github.com/anthropics/skills) | Official skills from Anthropic | `cd ~/.claude/skills && git clone https://github.com/anthropics/skills.git anthropic` |

### Community Skills

| Skill Repository | Description | Install Command |
|-----------------|-------------|-----------------|
| [Awesome Claude Skills](https://github.com/travisvn/awesome-claude-skills) | Curated collection of community skills | `cd ~/.claude/skills && git clone https://github.com/travisvn/awesome-claude-skills.git community` |
| [ComposioHQ Skills](https://github.com/ComposioHQ/awesome-claude-skills) | Another curated skill collection | `cd ~/.claude/skills && git clone https://github.com/ComposioHQ/awesome-claude-skills.git composio` |

### How to Install

**Option 1: Use `/update-skills`** (Recommended)
- Run `/update-skills` and it will check which recommended skills you're missing
- It will offer to install them for you

**Option 2: Manual Install**
- Copy the install command from the table above
- Skills from all directories in `~/.claude/skills/` are automatically available!

### Updating Public Skills

Public skills don't auto-update. Update manually:

```bash
cd ~/.claude/skills/anthropic && git pull
cd ~/.claude/skills/community && git pull
```

Or use `/update-skills` to update all skills at once.

## 📝 Contributing

1. Create a new branch for your skill
2. Add your skill in its own directory with a `SKILL.md` file
3. Update this README with skill documentation
4. Submit a PR for team review

## 📚 Skill Structure

Each skill should have:
- `SKILL.md` - The skill definition
- `README.md` - Documentation and examples (optional)
- `examples/` - Example usage (optional)

## 🔗 Resources

- [Claude Skills Documentation](https://code.claude.com/docs/en/skills)
- [Anthropic Skills Repository](https://github.com/anthropics/skills)
- [Skills Best Practices](https://zackproser.com/blog/claude-skills-internal-training)
