---
name: pdd-project-setup
description: Automate PDD project setup by creating Slack channels (tier-based) and copying Google Drive folder templates. Asks for region, city, building code, BudgetForce record ID, tier, and project type.
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
   - `restricted_template_id` — ID of the Restricted folder template
   - `region_drives{}` — mapping of region to shared drive ID
   - `tier_channels{}` — additional channels per tier (beyond the 3 base channels)
   - `regions[]` — valid region options

If the config file is missing or malformed, stop and tell the user to check the installation.

If any `region_drives` values are still `PLACEHOLDER_UPDATE_ME`, warn the user that Drive folder creation will be skipped for those regions.

## Phase 1 — Intake

Ask **all questions in a single `AskUserQuestion` batch**:

| # | Field | Question | Options |
|---|---|---|---|
| 1 | **Region** | "Which region is this project in?" | AMER, EMEA, JAPAC, LATAM, India |
| 2 | **City** | "What city is this project in? (e.g., New York, San Francisco, London)" | Text input |
| 3 | **Building Code** | "What is the building code? (e.g., NYC01, SFO02)" | Text input |
| 4 | **BudgetForce Record** | "What is the BudgetForce record name/ID?" | Text input |
| 5 | **Tier** | "What tier is this project?" | 1, 2, 3, 4, 5 |
| 6 | **Project Type** | "What type of project is this?" | Options from config `project_types` |

Parse all answers before proceeding.

**Normalization:**
- Region: uppercase for display (e.g., `AMER`, `EMEA`), lowercase for channel names
- City: Title case for display (e.g., `New York`, `San Francisco`)
- Building Code: uppercase for display, lowercase for channel names
- BudgetForce Record: preserve original case for display, lowercase + hyphenate for channel names

**Exit:** `{region, city, building_code, budgetforce_record, tier, project_type, drive_template_id, region_drive_id}`

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

## Phase 4 — Create Google Drive Folder Structure

Skip this phase if the region's drive ID is still `PLACEHOLDER_UPDATE_ME` in the config.

**Target structure:**
```
[Region Shared Drive]
└── [City Name]  <-- Check if exists, create if not
    └── [City], [Building Code], [BudgetForce Record Name]  <-- Create this
        ├── Non Restricted  <-- Copy GOE or OPS template here
        └── Restricted  <-- Copy Restricted template here
```

### Step 4.1 — Find or create City folder

1. List all folders in the region's shared drive root:
   ```
   list_drive_items(folder_id=<region_drive_id>, drive_id=<region_drive_id>, file_type="folder")
   ```

2. Search for a folder matching the city name (case-insensitive match).

3. If not found, create it:
   ```
   create_drive_folder(folder_name=<city_name>, parent_folder_id=<region_drive_id>)
   ```

**Exit:** `city_folder_id`

### Step 4.2 — Create project folder

Inside the city folder, create the project folder:
```
folder_name = "{City}, {Building Code}, {BudgetForce Record Name}"
create_drive_folder(folder_name=<folder_name>, parent_folder_id=<city_folder_id>)
```

**Exit:** `project_folder_id`

### Step 4.3 — Copy Non Restricted template

1. Create "Non Restricted" folder inside project folder:
   ```
   create_drive_folder(folder_name="Non Restricted", parent_folder_id=<project_folder_id>)
   ```

2. Get the template ID based on project type (GOE or OPS) from config.

3. List all items in the template folder:
   ```
   list_drive_items(folder_id=<drive_template_id>)
   ```

4. Copy each item (files and subfolders) into the "Non Restricted" folder:
   ```
   copy_drive_file(file_id=<item_id>, parent_folder_id=<non_restricted_folder_id>, new_name=<item_name>)
   ```
   Run all copies in parallel.

**Exit:** `non_restricted_folder_id`

### Step 4.4 — Copy Restricted template

1. Create "Restricted" folder inside project folder:
   ```
   create_drive_folder(folder_name="Restricted", parent_folder_id=<project_folder_id>)
   ```

2. Get the Restricted template ID from config (`restricted_template_id`).

3. List all items in the Restricted template folder:
   ```
   list_drive_items(folder_id=<restricted_template_id>)
   ```

4. Copy each item into the "Restricted" folder:
   ```
   copy_drive_file(file_id=<item_id>, parent_folder_id=<restricted_folder_id>, new_name=<item_name>)
   ```
   Run all copies in parallel.

**Exit:** `restricted_folder_id`

### Step 4.5 — Get shareable link

Get the link to the main project folder:
```
get_drive_shareable_link(file_id=<project_folder_id>)
```

**Exit:** `{project_folder_id, project_folder_link}`

## Phase 5 — Post Welcome Messages

For each created channel, post a welcome message with:
- Project details (region, building code, tier)
- Link to the Google Drive folder
- Link to BudgetForce record (if URL pattern is known)

Example message:
```markdown
**Welcome to the {City} {Building Code} Project Channel!**

📋 **Project Details:**
- Region: {region}
- City: {city}
- Building: {building_code}
- BudgetForce Record: {budgetforce_record}
- Tier: {tier}
- Project Type: {project_type}

📁 **Google Drive Folder:** {project_folder_link}

This channel was auto-created by the PDD Project Setup skill.
```

Use `slack_send_message(channel_id=<channel_id>, message=<message>)` for each channel.

Run all messages in parallel.

## Phase 6 — Summary

Print a completion summary:
```
✅ PDD Project Setup Complete

Project: {City}, {Building Code}, {BudgetForce Record}
Region: {Region} | Tier: {Tier} | Type: {Project Type}

Slack Channels Created:
  ✓ int-proj-amer-nyc01-bf-2026-q3-nyc → https://salesforce.slack.com/archives/C...
  ✓ proj-amer-nyc01-acct-bf-2026-q3-nyc → https://salesforce.slack.com/archives/C...
  [+ any tier-specific channels]

Manual Creation Required (External Workspace):
  ⚠ ext-proj-amer-nyc01-bf-2026-q3-nyc
    Instructions: Go to Salesforce External Slack, create channel with this exact name

Google Drive Folder Structure Created:
  📁 {City}/{City, Building Code, BudgetForce Record}/
     ├── Non Restricted (GOE/OPS template copied)
     └── Restricted (template copied)
  🔗 {project_folder_link}

Next Steps:
  1. Create the external channel manually in Salesforce External Slack
  2. Add team members to all channels
  3. Pin important links in each channel
  4. Review and customize the Drive folder structure
```

## Do / Don't

**Do**
- Always load fresh config from the templates.json file — never use hardcoded values.
- Normalize input appropriately: lowercase + hyphens for channel names, preserve case for display and folder names.
- Show the full channel plan and get approval before creating anything.
- Check if the city folder already exists before creating a new one (case-insensitive match).
- Run channel creation, folder copying, and message posting in parallel for speed.
- Copy all items from templates (files and subfolders) recursively.
- Provide clear instructions for manual external channel creation.

**Don't**
- Don't create channels until the user approves the plan.
- Don't fail silently if a channel or folder already exists — log it and continue.
- Don't attempt to create channels in the external workspace via MCP.
- Don't skip the welcome messages — they provide context for new team members.
- Don't create duplicate city folders — always search first.
