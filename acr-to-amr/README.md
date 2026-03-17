# ACR to AMR Migration Skill

An [Open Agent Skill](https://agentskills.io/) to help users migrate from **Azure Cache for Redis (ACR)** Basic/Standard/Premium tiers to **Azure Managed Redis (AMR)**.

## Overview

This skill assists AI agents in helping users:

- Compare features between ACR and Azure Managed Redis
- Select appropriate AMR SKUs based on existing cache configurations
- Pull cache metrics for data-driven SKU sizing
- Plan and execute migrations with best practices
- Get real-time pricing comparisons between ACR and AMR SKUs
- Troubleshoot common migration issues

**Supported source SKUs**: Basic (C0–C6), Standard (C0–C6), Premium (P1–P5)

> **Note**: For Enterprise/Enterprise Flash (ACRE) migration, see the sibling [`acre-to-amr-migration-skill`](../acre-to-amr/).

## Installation

### With GitHub Copilot CLI

#### Option A: Using the Vercel Skills CLI (`npx skills`)

```bash
npx skills add https://github.com/AzureManagedRedis/amr-migration-skill/acr-to-amr -g -a github-copilot
```

> ⚠️ When prompted, choose **"Copy to all agents"** instead of symlink.

Then reload and verify:

```
/skills reload
/skills info acr-to-amr-migration-skill
```

#### Option B: Clone and Copy

```bash
git clone https://github.com/AzureManagedRedis/amr-migration-skill
```

**Windows (PowerShell):**

```powershell
New-Item -ItemType Directory -Path "$env:USERPROFILE\.copilot\skills\acr-to-amr-migration-skill" -Force
Copy-Item -Path ".\amr-migration-skill\acr-to-amr\*" -Destination "$env:USERPROFILE\.copilot\skills\acr-to-amr-migration-skill\" -Recurse
```

**macOS / Linux:**

```bash
mkdir -p ~/.copilot/skills/acr-to-amr-migration-skill
cp -r ./amr-migration-skill/acr-to-amr/* ~/.copilot/skills/acr-to-amr-migration-skill/
```

Then reload and verify:

```
/skills reload
/skills info acr-to-amr-migration-skill
```

## Example Prompts

### Migration Planning
- "I have a Standard C3 cache in westus2. Help me migrate to Azure Managed Redis."
- "We're running a Premium P2 with 3 shards and geo-replication. What's the best AMR SKU?"

### SKU Selection & Pricing
- "Compare AMR SKU options for a workload using 10GB of memory with high server load."
- "What's the monthly cost difference between Standard C2 and the equivalent AMR SKU in eastus?"

### Feature Comparison
- "What Redis features does AMR support that ACR doesn't? We need JSON and Search."
- "Does AMR support VNet injection? We currently use that on our Premium cache."

### Metrics & Assessment
- "Pull metrics from my ACR cache `my-cache` in resource group `my-rg` and recommend an AMR SKU."
- "Assess our current cache usage and give me a full migration plan with pricing."

### Retirement & Timeline
- "When is Azure Cache for Redis being retired?"

## Folder Structure

```
acr-to-amr/
├── SKILL.md                          # Skill definition and migration workflow
├── README.md                         # This file
├── TODO.md                           # Roadmap items
├── references/
│   ├── amr-sku-specs.md              # AMR SKU definitions (M, B, X, Flash)
│   ├── azure-cli-commands.md         # Azure CLI reference for ACR discovery
│   ├── feature-comparison.md         # ACR vs AMR feature matrix
│   ├── mcp-server-config.md          # Microsoft Learn MCP server setup
│   ├── migration-overview.md         # Migration strategies and guidance
│   ├── pricing-tiers.md              # Pricing calculation rules (HA, shards, MRPP)
│   ├── retirement-faq.md             # ACR retirement dates and FAQ
│   └── sku-mapping.md                # SKU selection guidelines & decision matrix
└── scripts/
    ├── get_acr_metrics.ps1           # Pull ACR metrics for SKU sizing
    ├── get_acr_metrics.sh
    ├── get_redis_price.ps1           # Real-time pricing with HA/shards/MRPP logic
    └── get_redis_price.sh
```

## License

MIT
