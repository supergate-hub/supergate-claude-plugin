# Supergate Claude Code Plugins

A shared plugin marketplace for the Supergate team.

## Structure

- **`/plugins`** - Team-shared plugins

| Plugin | Description | Version |
|--------|-------------|---------|
| [work-logger](./plugins/work-logger/) | Collect and organize daily work activities into structured logs | 0.1.0 |

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

**Usage:**
```bash
/daily-work-logger            # Yesterday's log
/daily-work-logger 2026-02-11 # Specific date
```

## Contributing

1. Create `plugins/<plugin-name>/` with `.claude-plugin/plugin.json`
2. Add skills, commands, or agents
3. Register in `.claude-plugin/marketplace.json`
4. Submit a PR

## Documentation

- [Claude Code Plugins](https://code.claude.com/docs/en/plugins)
- [Plugin Marketplaces](https://code.claude.com/docs/en/plugin-marketplaces)
