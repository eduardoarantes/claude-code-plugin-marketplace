# Claude Code Plugin Marketplace

A marketplace for Claude Code plugins focused on production-ready Python development.

## Installation

### Adding the Marketplace

To add this marketplace to your Claude Code installation, use one of the following methods:

**Option 1: From a local directory**

If you've cloned this repository locally:

```bash
/plugin marketplace add /path/to/claude-code-plugin-marketplace
```

Replace `/path/to/claude-code-plugin-marketplace` with the absolute path to the directory where you cloned this repository.

**Option 2: From GitHub (when published)**

```bash
/plugin marketplace add owner/repo
```

Replace `owner/repo` with the GitHub repository path.

### Important Notes

- Point to the **directory** (not the marketplace.json file directly)
- Claude Code will automatically look for `.claude-plugin/marketplace.json` in that directory
- Plugin source paths in the marketplace are resolved relative to the marketplace directory root

## Plugin Installation Scope

You can install plugins at different scopes depending on your needs:

### Global Installation (User-Level)

Install plugins globally for all your projects:

```bash
/plugin install plugin-name@marketplace-name
```

When prompted, select **"User settings"**. This saves to `~/.claude/settings.json` and applies across all projects.

### Project-Level Installation (Recommended for Teams)

Install plugins for a specific project only:

**Option 1: Using the `/plugin` command**

```bash
/plugin install plugin-name@marketplace-name
```

When prompted, select **"Project"**. This saves to `.claude/settings.json` in your project root.

**Option 2: Manual configuration**

Create or edit `.claude/settings.json` in your project root:

```json
{
  "enabledMarketplaces": [
    {
      "name": "code-plugin-marketplace",
      "source": "./.claude-plugin/marketplace.json"
    }
  ],
  "enabledPlugins": {
    "python-coding-agents@code-plugin-marketplace": true,
    "voice-notification@code-plugin-marketplace": true
  }
}
```

**Benefits of project-level installation:**
- Plugins only active in specific projects
- Can commit `.claude/settings.json` to share with team
- Different projects can use different plugin configurations
- No conflicts between project requirements

### Local Overrides (Machine-Specific)

For machine-specific settings that shouldn't be committed:

Create `.claude/settings.local.json` in your project root. This file overrides both global and project settings but stays out of version control.

### Configuration Hierarchy

Settings are applied in this order (later overrides earlier):
1. **Global:** `~/.claude/settings.json` - Your personal defaults
2. **Project:** `.claude/settings.json` - Team shared settings
3. **Local:** `.claude/settings.local.json` - Machine-specific overrides

## Available Plugins

### python-coding-agents

A series of agents to support production-ready Python development.

**Installation:**

```bash
/plugin install python-coding-agents@code-plugin-marketplace
```

**Features:**
- Task implementation and review agents
- Code complexity analysis
- Test-driven development support
- Architecture planning
- Python expert code review

### voice-notification

Voice notifications when Claude needs user input, with support for multiple TTS providers.

**Installation:**

```bash
/plugin install voice-notification@code-plugin-marketplace
```

**Features:**
- Voice notifications via TTS when Claude needs input
- Automatic TTS provider detection (ElevenLabs, OpenAI, or pyttsx3)
- Personalized notifications with your name (30% of the time)
- Notification hook that triggers automatically

**Configuration:**

Optional environment variables:
- `ELEVENLABS_API_KEY` - For ElevenLabs TTS (highest quality)
- `OPENAI_API_KEY` - For OpenAI TTS (high quality)
- `ENGINEER_NAME` - Your name for personalized notifications (e.g., "Eduardo")
- No API keys needed - falls back to pyttsx3 (offline TTS)

## Verifying Installation

After adding the marketplace, verify it's configured:

```bash
/plugin marketplace list
```

Check available plugins:

```bash
/plugin list
```

Manage installed plugins:

```bash
/plugin
```

## Directory Structure

```
claude-code-plugin-marketplace/
├── .claude-plugin/
│   └── marketplace.json          # Marketplace configuration
├── python-coding-agents/          # Python development agents plugin
│   ├── .claude-plugin/
│   │   └── plugin.json           # Plugin metadata
│   ├── agents/                   # Agent definitions
│   └── commands/                 # Custom commands
├── voice-notification/            # Voice notification plugin
│   ├── plugin.json               # Plugin metadata with hooks
│   ├── notification.py           # Main notification script
│   └── utils/                    # TTS and LLM utilities
│       ├── tts/                  # TTS providers
│       └── llm/                  # LLM integrations
└── README.md
```

## Troubleshooting

### Marketplace installs but plugins not found

Make sure you're pointing to the marketplace **directory**, not the marketplace.json file:

❌ Wrong: `/plugin marketplace add ./.claude-plugin/marketplace.json`
✅ Correct: `/plugin marketplace add /absolute/path/to/marketplace`

### Plugin directory not found error

Verify that:
1. The plugin directory exists at the root level (e.g., `./python-coding-agents`)
2. The source path in marketplace.json starts with `./`
3. The plugin has a `.claude-plugin/plugin.json` file

## Contributing

To add new plugins to this marketplace:

1. Create your plugin directory at the root level
2. Add a `.claude-plugin/plugin.json` file with required metadata
3. Update `.claude-plugin/marketplace.json` to include your plugin:

```json
{
  "name": "your-plugin-name",
  "source": "./your-plugin-directory",
  "description": "Brief description of your plugin"
}
```

## Owner

**Eduardo Rodrigues**

## License

See individual plugin directories for license information.
