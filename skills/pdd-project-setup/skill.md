---
name: pdd-project-setup
description: Automate PDD project setup by creating Slack channels (tier-based) and copying Google Drive folder templates. Asks for region, building code, BudgetForce record ID, tier, and project type.
---

# PDD Project Setup

Automate project setup by creating standardized Slack channels and Google Drive folder structure.

## Phase 0 — Load configuration

1. Read the configuration file from this skill's repository:
   ```bash
   cat ~/.claude/plugins/cache/aisuite/pdd-project-setup/config/templates.json
   ```
2. Parse the JSON to get:
   - `project_types{}` — mapping of project type to Drive template folder ID
   - `tier_channels{}` — additional channels per tier (beyond the 3 base channels)
   - `regions[]` — valid region options

If the config file is missing or malformed, stop and tell the user to check the installation.

## Phase 1 — Intake

Ask **all questions in a single `AskUserQuestion` batch**:

| # | Field | Question | Options |
|---|---|---|---|
| 1 | **Region** | "Which region is this project in?" | AMER, EMEA, JAPAC, LATAM, India |
| 2 | **Building Code** | "What is the building code? (e.g., NYC01, SFO02)" | Text input |
| 3 | **BudgetForce Record** | "What is the BudgetForce record ID?" | Text input |
| 4 | **Tier** | "What tier is this project?" | 1, 2, 3, 4, 5 |
| 5 | **Project Type** | "What type of project is this?" | Options from config `project_types` |

Parse all answers before proceeding.

**Normalization:**
- Region: lowercase (e.g., `amer`, `emea`, `japac`, `latam`, `india`)
- Building Code: lowercase
- BudgetForce Record: lowercase, spaces replaced with hyphens

**Exit:** `{region, building_code, budgetforce_record, tier, project_type, drive_template_id}`

## Phase 2 — Channel Planning

Build a list of channels to create based on tier.

### Base Channels (All Tiers)

1. `ext-proj-{region}-{building_code}-{budgetforce_record}` — **External workspace (manual)**
2. `int-proj-{region}-{building_code}-{budgetforce_record}` — Internal workspace
3. `proj-{region}-{building_code}-acct-{budgetforce_record}` — Internal workspace

### Tier-Specific Channels

Look up `tier_channels[tier]` from the config. For each additional channel pattern:
- Substitute `{region}`, `{building_code}`, `{budgetforce_record}` into the pattern
- Add to the channel list

**Show the user the full channel plan:**
```
Channels to create:
  ✓ int-proj-amer-nyc01-bf-2026-q3-nyc (internal, will be created)
  ✓ proj-amer-nyc01-acct-bf-2026-q3-nyc (internal, will be created)
  ⚠ ext-proj-amer-nyc01-bf-2026-q3-nyc (external, manual creation required)
  [+ any tier-specific channels]

Google Drive folder template: [project_type description]
```

Ask: **proceed / cancel**. Don't continue until approved.

**Exit:** approved `channels_to_create[]` and `manual_channels[]`

## Phase 3 — Create Slack Channels

For each channel in `channels_to_create[]` (internal workspace channels only):

1. Create the channel:
   ```
   slack_create_conversation(channel_name=<channel_name>, is_private=false)
   ```
   Log the returned `channel_id`.

2. If creation fails with `name_taken`, log it as "already exists" and try to search for the channel ID:
   ```
   slack_search_channels(query=<channel_name>)
   ```

3. If creation fails with `restricted_action` or other permission error, log it and tell the user they may need admin assistance.

Run all channel creations in parallel.

**Exit:** `created_channels[]` with `{name, channel_id}` for each

## Phase 4 — Copy Google Drive Template

1. Get the Drive template folder ID from `drive_template_id` (from config based on project type).

2. Copy the template folder:
   ```
   copy_drive_file(file_id=<drive_template_id>, new_name="[Project] {region}-{building_code}-{budgetforce_record}")
   ```
   Note: If the API doesn't support folder copying, fall back to:
   - Create a new folder: `create_drive_folder(folder_name="[Project] {region}-{building_code}-{budgetforce_record}")`
   - List items in template: `list_drive_items(folder_id=<drive_template_id>)`
   - Copy each item into the new folder

3. Get the shareable link:
   ```
   get_drive_shareable_link(file_id=<new_folder_id>)
   ```

**Exit:** `{drive_folder_id, drive_folder_link}`

## Phase 5 — Post Welcome Messages

For each created channel, post a welcome message with:
- Project details (region, building code, tier)
- Link to the Google Drive folder
- Link to BudgetForce record (if URL pattern is known)

Example message:
```markdown
**Welcome to the [Region] [Building Code] Project Channel!**

📋 **Project Details:**
- Region: {region}
- Building: {building_code}
- BudgetForce Record: {budgetforce_record}
- Tier: {tier}
- Project Type: {project_type}

📁 **Google Drive Folder:** {drive_folder_link}

This channel was auto-created by the PDD Project Setup skill.
```

Use `slack_send_message(channel_id=<channel_id>, message=<message>)` for each channel.

Run all messages in parallel.

## Phase 6 — Summary

Print a completion summary:
```
✅ PDD Project Setup Complete

Slack Channels Created:
  ✓ int-proj-amer-nyc01-bf-2026-q3-nyc → https://salesforce.slack.com/archives/C...
  ✓ proj-amer-nyc01-acct-bf-2026-q3-nyc → https://salesforce.slack.com/archives/C...

Manual Creation Required (External Workspace):
  ⚠ ext-proj-amer-nyc01-bf-2026-q3-nyc
    Instructions: Go to Salesforce External Slack, create channel with this exact name

Google Drive Folder:
  📁 {drive_folder_link}

Next Steps:
  1. Create the external channel manually
  2. Add team members to all channels
  3. Pin important links in each channel
```

## Do / Don't

**Do**
- Always load fresh config from the templates.json file — never use hardcoded values.
- Normalize all input to lowercase and replace spaces with hyphens for channel names.
- Show the full channel plan and get approval before creating anything.
- Run channel creation and message posting in parallel for speed.
- Provide clear instructions for manual external channel creation.

**Don't**
- Don't create channels until the user approves the plan.
- Don't fail silently if a channel already exists — log it and continue.
- Don't attempt to create channels in the external workspace via MCP.
- Don't skip the welcome messages — they provide context for new team members.
