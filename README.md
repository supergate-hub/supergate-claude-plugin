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
- `summarize-pdf` - Read a technical book PDF chapter by chapter and create structured Obsidian notes
- `translate-article` - Translate a technical article URL (full translation, no summarization)
- `translate-youtube` - Translate a YouTube transcript (full translation, no summarization)
- `github-project` - Analyze a GitHub repository and create an Obsidian project document
- `leetcode-review` - Convert LeetCode problem review conversations into structured Obsidian notes
- `publish-confluence` - Publish an Obsidian document to Confluence with Mermaid diagrams and images

**Usage:**
```bash
/summarize-article https://example.com/article
/summarize-youtube https://youtube.com/watch?v=xxx
/summarize-youtube kr https://youtube.com/watch?v=xxx  # Korean transcript preferred
/summarize-pdf ~/Downloads/book.pdf        # Show TOC and select chapters
/summarize-pdf ~/Downloads/book.pdf 3      # Process chapter 3 only
/summarize-pdf ~/Downloads/book.pdf 1-5   # Process chapters 1-5
/summarize-pdf ~/Downloads/book.pdf p.45-80 # Process specific page range
/translate-article https://example.com/article
/translate-youtube https://youtube.com/watch?v=xxx
/github-project https://github.com/owner/repo
/leetcode-review               # Auto-detect from current conversation
/leetcode-review 190           # By problem number (uses repo code)
/publish-confluence "Clean Architecture - Chapter 3"       # Publish to Confluence (asks for space/parent)
/publish-confluence "API 설계 가이드" DEV 1228177436        # With space key and parent page ID
```

## Contributing

1. Create `plugins/<plugin-name>/` with `.claude-plugin/plugin.json`
2. Add skills, commands, or agents
3. Register in `.claude-plugin/marketplace.json`
4. Submit a PR

## Documentation

- [Claude Code Plugins](https://code.claude.com/docs/en/plugins)
- [Plugin Marketplaces](https://code.claude.com/docs/en/plugin-marketplaces)
