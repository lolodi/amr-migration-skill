# AMR Migration Skills

Two [Open Agent Skills](https://agentskills.io) to help users migrate to **Azure Managed Redis (AMR)** from Azure Cache for Redis products.

## Skills

| Skill | Source | Target | Directory |
|-------|--------|--------|-----------|
| **`acr-to-amr-migration-skill`** | ACR Basic/Standard/Premium (`Microsoft.Cache/redis`) | AMR | [`acr-to-amr/`](acr-to-amr/) |
| **`acre-to-amr-migration-skill`** | ACRE Enterprise/Enterprise Flash (`Microsoft.Cache/redisEnterprise`) | AMR | [`acre-to-amr/`](acre-to-amr/) |

These are **independent skills** with separate triggering, reference files, and migration logic. Each skill includes a scope boundary that redirects to the other when the user's request doesn't match.

## Installation

### With GitHub Copilot CLI

#### Option A: Using the Vercel Skills CLI (`npx skills`)

```bash
# Install both skills
npx skills add https://github.com/AzureManagedRedis/amr-migration-skill/acr-to-amr -g -a github-copilot
npx skills add https://github.com/AzureManagedRedis/amr-migration-skill/acre-to-amr -g -a github-copilot
```

> ⚠️ When prompted for the installation method, choose **"Copy to all agents"**. On Windows, symlink creation requires Administrator privileges or Developer Mode.

#### Option B: Clone and Copy

```bash
git clone https://github.com/AzureManagedRedis/amr-migration-skill
```

**Windows (PowerShell):**

```powershell
# ACR skill
New-Item -ItemType Directory -Path "$env:USERPROFILE\.copilot\skills\acr-to-amr-migration-skill" -Force
Copy-Item -Path ".\amr-migration-skill\acr-to-amr\*" -Destination "$env:USERPROFILE\.copilot\skills\acr-to-amr-migration-skill\" -Recurse

# ACRE skill
New-Item -ItemType Directory -Path "$env:USERPROFILE\.copilot\skills\acre-to-amr-migration-skill" -Force
Copy-Item -Path ".\amr-migration-skill\acre-to-amr\*" -Destination "$env:USERPROFILE\.copilot\skills\acre-to-amr-migration-skill\" -Recurse
```

**macOS / Linux:**

```bash
# ACR skill
mkdir -p ~/.copilot/skills/acr-to-amr-migration-skill
cp -r ./amr-migration-skill/acr-to-amr/* ~/.copilot/skills/acr-to-amr-migration-skill/

# ACRE skill
mkdir -p ~/.copilot/skills/acre-to-amr-migration-skill
cp -r ./amr-migration-skill/acre-to-amr/* ~/.copilot/skills/acre-to-amr-migration-skill/
```

Then reload and verify:

```
/skills reload
/skills info acr-to-amr-migration-skill
/skills info acre-to-amr-migration-skill
```

### With Claude Code

Add each skill directory to your agent skills directory.

## Repository Structure

```
amr-migration-skill/
├── README.md                         # This file
├── LICENSE
├── acr-to-amr/                       # ACR → AMR migration skill
│   ├── SKILL.md                      # Skill definition
│   ├── README.md                     # ACR skill overview and prompts
│   ├── TODO.md                       # Roadmap
│   ├── references/                   # ACR-specific reference files
│   │   ├── amr-sku-specs.md
│   │   ├── azure-cli-commands.md
│   │   ├── feature-comparison.md
│   │   ├── mcp-server-config.md
│   │   ├── migration-overview.md
│   │   ├── pricing-tiers.md
│   │   ├── retirement-faq.md
│   │   └── sku-mapping.md
│   └── scripts/
│       ├── get_acr_metrics.ps1
│       ├── get_acr_metrics.sh
│       ├── get_redis_price.ps1
│       └── get_redis_price.sh
└── acre-to-amr/                      # ACRE → AMR migration skill
    ├── SKILL.md                      # Skill definition (3 modes)
    ├── README.md                     # ACRE skill overview and prompts
    ├── TODO.md                       # Roadmap
    ├── references/                   # ACRE-specific reference files
    │   ├── breaking-changes.md
    │   ├── checklists.md
    │   ├── examples.md
    │   ├── interactive-migration.md
    │   ├── migration-guide.md
    │   ├── script-analysis.md
    │   ├── sku-resolution.md
    │   └── troubleshooting.md
    └── scripts/
        ├── get_redis_price.ps1
        └── get_redis_price.sh
```

## Example Prompts

### ACR → AMR (Basic/Standard/Premium)
- "I have a Standard C3 cache in westus2. Help me migrate to AMR."
- "Compare AMR SKU options for my Premium P2 with 3 shards."
- "What's the monthly cost difference between Standard C2 and the equivalent AMR SKU?"
- "Pull metrics from my ACR cache and recommend an AMR SKU."

### ACRE → AMR (Enterprise/Enterprise Flash)
- "I have an ARM template for an ACRE cache — what changes for AMR?"
- "Help me migrate my ACRE caches to AMR step by step."
- "What is the recommended process to migrate from ACRE to AMR?"
- "Review my PowerShell script and tell me what breaks after ACRE to AMR migration."

## Contributing

1. Keep documentation up-to-date with latest Azure Redis features
2. Update SKU mappings when new AMR SKUs are released
3. Maintain clear scope boundaries between the two skills

## License

MIT
