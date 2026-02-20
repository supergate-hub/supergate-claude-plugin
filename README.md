# Supergate Claude Code Plugins

A shared plugin marketplace for the Supergate team.

## Structure

- **`/plugins`** - Team-shared plugins

| Plugin | Description | Version |
|--------|-------------|---------|
| [work-logger](./plugins/work-logger/) | Collect and organize daily work activities into structured logs | 0.1.0 |
| [obsidian](./plugins/obsidian/) | Capture external knowledge into structured Obsidian documents | 0.1.0 |

## Installation

```bash
# Add marketplace
/plugin marketplace add supergate-hub/supergate-claude-plugin

# Install a plugin
/plugin install work-logger@supergate-claude-plugin
```

## Available Plugins

### work-logger

Collects work activities from multiple sources and generates structured daily reports.

**Included skills:**
- `daily-work-logger` - Analyze yesterday's work from Obsidian vault, Claude sessions, and meeting notes
- `learning-tracker` - Extract new tech/library/concept learnings from sessions into TIL documents

**Usage:**
```bash
/daily-work-logger            # Yesterday's log
/daily-work-logger 2026-02-11 # Specific date
/learning-tracker             # Yesterday's learnings
/learning-tracker 2026-02-11  # Specific date
```

### obsidian

Capture external knowledge from articles, YouTube videos, and GitHub repositories into structured Obsidian documents.

**Included commands:**
- `summarize-article` - Summarize and translate a technical article URL into an Obsidian note
- `summarize-youtube` - Summarize and translate a YouTube video into an Obsidian note
- `translate-article` - Translate a technical article URL (full translation, no summarization)
- `translate-youtube` - Translate a YouTube transcript (full translation, no summarization)
- `github-project` - Analyze a GitHub repository and create an Obsidian project document
- `leetcode-review` - Convert LeetCode problem review conversations into structured Obsidian notes

**Usage:**
```bash
/summarize-article https://example.com/article
/summarize-youtube https://youtube.com/watch?v=xxx
/summarize-youtube kr https://youtube.com/watch?v=xxx  # Korean transcript preferred
/translate-article https://example.com/article
/translate-youtube https://youtube.com/watch?v=xxx
/github-project https://github.com/owner/repo
/leetcode-review               # Auto-detect from current conversation
/leetcode-review 190           # By problem number (uses repo code)
```

## Contributing

1. Create `plugins/<plugin-name>/` with `.claude-plugin/plugin.json`
2. Add skills, commands, or agents
3. Register in `.claude-plugin/marketplace.json`
4. Submit a PR

## Documentation

- [Claude Code Plugins](https://code.claude.com/docs/en/plugins)
- [Plugin Marketplaces](https://code.claude.com/docs/en/plugin-marketplaces)
