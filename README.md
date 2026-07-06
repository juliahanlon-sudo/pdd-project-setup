# PDD Project Setup

Claude Code skill for automating PDD project setup — creates Slack channels, copies Google Drive folder templates, and posts welcome messages.

## Features

- Creates standardized Slack channels based on region, building code, and BudgetForce record
- Tier-based channel creation (Tiers 1-5 with tier-specific channels)
- Project-type-specific Google Drive folder templates
- Automated welcome messages to channels
- Support for multiple regions: AMER, EMEA, JAPAC, LATAM, India

## Installation

1. Clone this repository into your Claude Code plugins directory:
   ```bash
   cd ~/.claude/plugins/cache/aisuite/
   git clone <repo-url> pdd-project-setup
   ```

2. Restart Claude Code or run `/skills` to reload

## Usage

Invoke the skill:
```
/pdd-project-setup
```

The skill will prompt you for:
- **Region:** AMER, EMEA, JAPAC, LATAM, or India
- **Building Code:** e.g., NYC01, SFO02
- **BudgetForce Record ID:** The BudgetForce record identifier
- **Tier:** 1, 2, 3, 4, or 5 (determines channel count)
- **Project Type:** Determines which Google Drive template to copy

## Channel Naming Convention

### Base Channels (All Tiers)
1. `ext-proj-[region]-[buildingcode]-[budgetforce-record]` — External workspace (manual creation required)
2. `int-proj-[region]-[buildingcode]-[budgetforce-record]` — Internal workspace
3. `proj-[region]-[buildingcode]-acct-[budgetforce-record]` — Internal workspace

### Tier-Specific Channels
- **Tiers 1, 2, 3:** Additional channels (to be defined)
- **Tiers 4, 5:** Base channels only

## Known Limitations

- External Slack workspace channels (`ext-*`) cannot be created via MCP — the skill provides instructions for manual creation
- Requires Slack MCP and Google Workspace MCP to be connected

## Configuration

Project type templates are configured in `config/templates.json`:
```json
{
  "project_types": {
    "type_name": {
      "drive_template_id": "1ABC...XYZ",
      "description": "Description of project type"
    }
  }
}
```

## Development

To update tier-specific channels, edit `skills/pdd-project-setup/skill.md` and modify the channel creation logic in Phase 2.

## License

Internal Salesforce use only.
