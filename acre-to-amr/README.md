# ACRE to AMR Migration Skill

An [Open Agent Skill](https://agentskills.io/) to help users migrate from Azure Cache for Redis Enterprise (ACRE) to Azure Managed Redis (AMR).

## Overview

This skill assists AI agents in helping users:

- Understand what changes are required in their automation scripts (ARM templates, Bicep templates, Azure CLI, Azure PowerShell) after migrating from ACRE to AMR
- Get step-by-step interactive guidance to migrate ACRE caches to AMR, including discovering caches, executing migration with confirmation gates, and handling geo-replicated cache scenarios
- Get a generic recommended step-by-step migration guide or flowchart for ACRE to AMR migration without executing any commands against a live subscription

**Supported source SKUs**: Enterprise (`Enterprise_*`), Enterprise Flash (`EnterpriseFlash_*`)

> **Note**: For Basic/Standard/Premium (ACR) migration, see the sibling [`acr-to-amr-migration-skill`](../acr-to-amr/).

## Installation

### With GitHub Copilot CLI

#### Option A: Using the Vercel Skills CLI (`npx skills`)

```bash
npx skills add https://github.com/AzureManagedRedis/amr-migration-skill/acre-to-amr -g -a github-copilot
```

> ⚠️ When prompted, choose **"Copy to all agents"** instead of symlink.

Then reload and verify:

```
/skills reload
/skills info acre-to-amr-migration-skill
```

#### Option B: Clone and Copy

```bash
git clone https://github.com/AzureManagedRedis/amr-migration-skill
```

**Windows (PowerShell):**

```powershell
New-Item -ItemType Directory -Path "$env:USERPROFILE\.copilot\skills\acre-to-amr-migration-skill" -Force
Copy-Item -Path ".\amr-migration-skill\acre-to-amr\*" -Destination "$env:USERPROFILE\.copilot\skills\acre-to-amr-migration-skill\" -Recurse
```

**macOS / Linux:**

```bash
mkdir -p ~/.copilot/skills/acre-to-amr-migration-skill
cp -r ./amr-migration-skill/acre-to-amr/* ~/.copilot/skills/acre-to-amr-migration-skill/
```

Then reload and verify:

```
/skills reload
/skills info acre-to-amr-migration-skill
```

## Example Prompts

Try these prompts to get started:

### Automation Script Update

- "I have an ARM template that deploys an ACRE cache — what do I need to change for AMR?"
- "Here's my Bicep template for Redis Enterprise. Can you give me a checklist of changes needed after migrating to AMR?"
- "My Azure CLI script creates an Enterprise_E10 cache with capacity 2. What's the equivalent AMR command?"
- "Review my PowerShell deployment script and tell me what breaks after ACRE to AMR migration."
- "What AMR SKU should I use to replace my Enterprise_E20 with capacity 4?"

### Interactive Cache Migration

- "Help me migrate my ACRE caches to AMR."
- "Discover all Redis Enterprise caches in my subscription and tell me which ones are still on ACRE."
- "Migrate my geo-replicated ACRE caches to AMR step by step."
- "I have ACRE caches with Private Link — walk me through the migration."
- "Which of my caches are eligible for in-place migration vs. create-new-and-swap?"

### Generic Migration Guide

- "What is the recommended step-by-step process to migrate from ACRE to AMR?"
- "Explain the difference between in-place migration and create-new-and-swap for geo-replicated caches."
- "What are the breaking changes between ACRE and AMR that I should know about?"
- "What happens to my Private Endpoints and DNS zones after migration?"
- "Is there any downtime during in-place migration? Can I roll back?"

## Folder Structure

```
acre-to-amr/
├── SKILL.md                          # Core: scope, mode detection, behavior rules, navigation index
├── README.md                         # This file
├── TODO.md                           # Roadmap items
├── references/
│   ├── breaking-changes.md           # Property changes, DNS updates, API versions, PE migration strategies
│   ├── script-analysis.md            # Detection patterns, companion files, Gen1 indicators, validation
│   ├── sku-resolution.md             # listSkusForScaling procedure + fallback mapping table + SKU families
│   ├── checklists.md                 # Per-script checklists (ARM, Bicep, CLI, PowerShell)
│   ├── examples.md                   # Before/after code examples for each script type
│   ├── migration-guide.md            # Generic migration guide (overview, flowchart, Path A/B summaries)
│   ├── interactive-migration.md      # Interactive migration (discovery + in-place + geo-replicated)
│   └── troubleshooting.md            # Troubleshooting & FAQ
└── scripts/
    ├── get_redis_price.ps1           # Real-time AMR pricing lookup
    └── get_redis_price.sh
```

## Per-Mode Reference Loading

Each mode only needs a subset of reference files, keeping agent context lean:

| Mode | Files Needed |
|------|-------------|
| **Automation Script Update** | SKILL.md → SCRIPT-ANALYSIS → SKU-RESOLUTION → CHECKLISTS → BREAKING-CHANGES → EXAMPLES |
| **Interactive Cache Migration** | SKILL.md → INTERACTIVE-MIGRATION → SKU-RESOLUTION |
| **Generic Migration Guide** | SKILL.md → MIGRATION-GUIDE → BREAKING-CHANGES |

## External Resources

This skill leverages:

- **Microsoft Learn MCP Server**: `https://learn.microsoft.com/api/mcp` for up-to-date Azure documentation
- **SKU Resolution Data**: `references/sku-resolution.md` for ACRE to AMR SKU mapping and fallback table
- **Azure CLI Reference**: For `az redisenterprise` and `az rest` commands used in migration

## Contributing

1. Keep documentation up-to-date with latest Azure Redis features
2. Update SKU mappings when new AMR SKUs are released
3. Add scripts for common migration automation tasks
4. Update SKILL.md when new migration patterns emerge

## License

MIT
