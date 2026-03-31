# VNET Cache Migration Support — Design Spec

## Problem Statement

The AMR migration skill currently treats VNET-injected caches as unsupported for automated migration. The automated migration validation block for VNET caches is being removed, but the skill has no guidance for the additional networking steps VNET users need (Private Endpoint setup on AMR). Users with VNET caches hit a dead end.

## Proposed Approach

Integrate VNET awareness organically into the existing migration workflow. The agent detects VNET status during assessment, adds Private Endpoint setup as a pre-step, and guides users through the complete migration including networking changes. No separate workflow path — just conditional guidance within the existing Steps 1–4.

## Key Facts

- AMR does not support VNet injection; Private Link (Private Endpoints) replaces it
- AMR Private DNS zone: `privatelink.redis.azure.net`
- OSS Private Endpoint caches remain excluded from automated migration (only VNET injection is unblocked)
- The agent can extract VNet/subnet info from the source cache's `subnetId` property

---

## Changes

### 1. SKILL.md — Agent Guidance

**1a. New "Detecting VNET Caches" section** (under Agent Guidance):
- When assessing a cache, check for `subnetId` in `az redis show` output
- If present → cache is VNET-injected → add PE setup to the migration plan
- CLI command: `az redis show -n <name> -g <rg> --query "{subnetId: subnetId}"`
- Extract VNet name and subnet from the subnetId to use for PE creation

**1b. Update Step 3 (Plan Migration)**:
- Add VNET conditional: "If VNET detected, create a Private Endpoint on the target AMR cache in the same VNet before migration. See private-endpoint-setup.md."
- Rewrite existing VNET language to be actionable rather than blocking

**1c. Update Automated Migration section**:
- Remove "VNet injected" from the exclusion list (keep "Private Link enabled" and "Geo-Replicated")
- Add: "For VNet-injected caches, additional networking setup (Private Endpoints) is required before migration."

**1d. Update Connection Changes Reminder**:
- Add VNET-specific note: VNet injection → Private Endpoint (different isolation model)
- Mention that NSG rules from VNet injection don't carry over

### 2. New Reference Doc — `references/private-endpoint-setup.md`

Contents:
1. **Overview** — Why PE is needed (AMR uses Private Link instead of VNet injection)
2. **Extracting VNet info from the source cache** — `az redis show` to get `subnetId`, parse VNet name/subnet
3. **Creating a Private Endpoint on the AMR target** — `az network private-endpoint create` in the same VNet
4. **Private DNS Zone setup** — Create/link `privatelink.redis.azure.net` zone in the VNet
5. **Validation** — Verify PE connectivity before migration
6. **Cleanup** — Post-migration, the old VNET-injected cache's subnet can be freed

### 3. Reference Doc Updates

**3a. `references/automated-migration.md`**:
- Remove "VNet injected caches" from unsupported list
- Add "VNET Cache Pre-requisites" section linking to private-endpoint-setup.md

**3b. `references/migration-validation.md`**:
- Remove the "VNet injection enabled on source" error entry
- Keep "Private endpoints on source" error (OSS PE caches remain unsupported)

**3c. `references/migration-overview.md`**:
- Update VNET section from "not supported" to "supported with additional networking steps"

**3d. `references/feature-comparison.md`**:
- Update VNET row to clarify migration path exists

**3e. `references/azure-cli-commands.md`**:
- Add VNET detection commands
- Add commands for extracting VNet/subnet details from subnetId

### 4. Eval Updates

**4a. Update eval ID 4**: Rework so VNET is no longer tested as a blocker — test that the agent detects VNET and recommends PE setup. Keep geo-replication and PE-on-source as blockers.

**4b. New eval — VNET migration success path**: Premium P2 with VNet injection → agent detects VNET, recommends SKU, guides PE setup, proceeds with automated migration.

**4c. New eval — VNET subnet lookup**: Tests that the agent extracts VNet/subnet info and uses it for PE creation guidance.
