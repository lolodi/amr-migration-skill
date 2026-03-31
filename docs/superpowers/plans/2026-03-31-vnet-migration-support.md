# VNET Cache Migration Support — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Enable the AMR migration skill to organically guide users through migrating VNET-injected caches, including Private Endpoint setup on the AMR target.

**Architecture:** Integrate VNET awareness into the existing migration workflow by (1) adding agent guidance to detect VNET caches, (2) creating a new Private Endpoint setup reference doc, (3) updating all existing reference docs to remove VNET-as-blocker language, and (4) adding eval coverage for the VNET migration path.

**Tech Stack:** Markdown reference docs, JSON eval definitions, Azure CLI commands

**Spec:** `docs/superpowers/specs/2026-03-31-vnet-migration-support-design.md`

---

### Task 1: Create `references/private-endpoint-setup.md`

**Files:**
- Create: `references/private-endpoint-setup.md`

This is the key new document. It provides Azure CLI commands for setting up Private Endpoints on AMR caches — the replacement for VNet injection.

- [ ] **Step 1: Create the file**

Create `references/private-endpoint-setup.md` with this content:

```markdown
# Private Endpoint Setup for AMR (VNET Migration)

When migrating a VNet-injected ACR cache to AMR, you must set up a **Private Endpoint** on the target AMR cache to maintain network isolation. AMR does not support VNet injection — Private Link (via Private Endpoints) is the replacement.

---

## Step 1: Extract VNet Info from the Source Cache

Get the subnet ID from the source ACR cache to identify which VNet/subnet to create the Private Endpoint in:

\`\`\`bash
az redis show -n <cacheName> -g <resourceGroup> --query "subnetId" -o tsv
\`\`\`

This returns a full ARM resource ID like:
\`\`\`
/subscriptions/<subId>/resourceGroups/<rg>/providers/Microsoft.Network/virtualNetworks/<vnetName>/subnets/<subnetName>
\`\`\`

Extract the VNet name and subnet name from this path — you'll use the same VNet (and optionally the same or a different subnet) for the Private Endpoint.

To also get the VNet's resource group (which may differ from the cache's resource group):

\`\`\`bash
# Parse the VNet resource group from the subnetId
SUBNET_ID=$(az redis show -n <cacheName> -g <resourceGroup> --query "subnetId" -o tsv)
VNET_RG=$(echo $SUBNET_ID | cut -d'/' -f5)
VNET_NAME=$(echo $SUBNET_ID | cut -d'/' -f9)
SUBNET_NAME=$(echo $SUBNET_ID | cut -d'/' -f11)
echo "VNet RG: $VNET_RG, VNet: $VNET_NAME, Subnet: $SUBNET_NAME"
\`\`\`

---

## Step 2: Create a Private Endpoint on the AMR Cache

Create a Private Endpoint in the same VNet as the source cache. Use a subnet that does **not** have the `Microsoft.Cache/Redis` service delegation (VNET-injected caches use a delegated subnet that cannot be shared with Private Endpoints).

> **Subnet note**: The subnet used for VNet injection has a service delegation to `Microsoft.Cache/Redis`. You can use a **different subnet** in the same VNet for the Private Endpoint. If needed, create a new subnet:
>
> \`\`\`bash
> az network vnet subnet create \
>   --resource-group <vnetResourceGroup> \
>   --vnet-name <vnetName> \
>   --name pe-subnet \
>   --address-prefix 10.0.2.0/24
> \`\`\`

Create the Private Endpoint:

\`\`\`bash
az network private-endpoint create \
  --name <amrCacheName>-pe \
  --resource-group <resourceGroup> \
  --vnet-name <vnetName> \
  --subnet <subnetName> \
  --private-connection-resource-id /subscriptions/<subId>/resourceGroups/<rg>/providers/Microsoft.Cache/redisEnterprise/<amrCacheName> \
  --group-id redisEnterprise \
  --connection-name <amrCacheName>-connection
\`\`\`

> **Parameters**:
> - `--private-connection-resource-id`: Full ARM resource ID of the **target AMR cache** (uses `Microsoft.Cache/redisEnterprise` provider)
> - `--group-id`: Must be `redisEnterprise` for AMR caches
> - `--subnet`: Use a subnet without the `Microsoft.Cache/Redis` delegation

---

## Step 3: Configure Private DNS Zone

Create a Private DNS zone and link it to the VNet so the AMR cache hostname resolves to the Private Endpoint's private IP:

\`\`\`bash
# Create the Private DNS zone (if it doesn't already exist)
az network private-dns zone create \
  --resource-group <resourceGroup> \
  --name privatelink.redis.azure.net

# Link the DNS zone to the VNet
az network private-dns zone vnet-link create \
  --resource-group <resourceGroup> \
  --zone-name privatelink.redis.azure.net \
  --name <vnetName>-link \
  --virtual-network /subscriptions/<subId>/resourceGroups/<vnetRg>/providers/Microsoft.Network/virtualNetworks/<vnetName> \
  --registration-enabled false

# Create a DNS record group linking to the Private Endpoint
az network private-endpoint dns-zone-group create \
  --resource-group <resourceGroup> \
  --endpoint-name <amrCacheName>-pe \
  --name default \
  --private-dns-zone /subscriptions/<subId>/resourceGroups/<rg>/providers/Microsoft.Network/privateDnsZones/privatelink.redis.azure.net \
  --zone-name privatelink.redis.azure.net
\`\`\`

> **DNS zone**: AMR uses `privatelink.redis.azure.net`. This is different from OSS ACR (`privatelink.redis.cache.windows.net`) and legacy Enterprise (`privatelink.redisenterprise.cache.azure.net`).

---

## Step 4: Validate Connectivity

Verify the Private Endpoint is working before proceeding with migration:

\`\`\`bash
# Check Private Endpoint status
az network private-endpoint show \
  --name <amrCacheName>-pe \
  --resource-group <resourceGroup> \
  --query "privateLinkServiceConnections[0].privateLinkServiceConnectionState.status" -o tsv
\`\`\`

Expected output: `Approved`

To verify DNS resolution from within the VNet (run from a VM in the same VNet):

\`\`\`bash
nslookup <amrCacheName>.<region>.redis.azure.net
\`\`\`

The result should resolve to a **private IP** address in the VNet's address range, not a public IP.

---

## Step 5: Post-Migration Cleanup

After migration completes and the old ACR cache is decommissioned:

1. **Remove the old cache**: `az redis delete -n <oldCacheName> -g <resourceGroup>`
2. **Free the delegated subnet**: Once the VNET-injected cache is deleted, the subnet delegation (`Microsoft.Cache/Redis`) is removed and the subnet can be repurposed or deleted
3. **Update NSG rules**: VNet injection NSG rules (ports 6379/6380, Redis cluster bus ports) are no longer needed. Private Endpoints use the standard Private Link security model

---

## Quick Reference

| Item | Value |
|------|-------|
| Private DNS zone | `privatelink.redis.azure.net` |
| Group ID | `redisEnterprise` |
| AMR resource provider | `Microsoft.Cache/redisEnterprise` |
| Subnet requirement | No `Microsoft.Cache/Redis` service delegation |
```

- [ ] **Step 2: Commit**

```bash
git add references/private-endpoint-setup.md
git commit -m "Add Private Endpoint setup reference for VNET cache migration"
```

---

### Task 2: Update SKILL.md — Agent Guidance

**Files:**
- Modify: `SKILL.md`

Add VNET detection guidance and update existing sections. Three edits needed.

- [ ] **Step 1: Add VNET detection section after Connection Changes Reminder (after line 74)**

Find this text in SKILL.md:
```
If the user is using the automated migration with DNS switching, the old hostname continues to work, but the port change still applies.
```

After it, add a new section:

```markdown

### Detecting VNET-Injected Caches
When assessing a cache for migration, always check whether it uses VNet injection:

```bash
az redis show -n <cacheName> -g <resourceGroup> --query "{subnetId: subnetId, privateEndpoints: privateEndpointConnections}" -o json
```

- If `subnetId` is **non-null**: the cache is **VNet-injected**. AMR does not support VNet injection, so a Private Endpoint must be configured on the target AMR cache before migration. Follow [Private Endpoint Setup](references/private-endpoint-setup.md).
- Extract the VNet name and subnet from the `subnetId` to help the user create the Private Endpoint in the same network.
- If `privateEndpointConnections` is **non-empty**: the source cache has Private Endpoints. Private Endpoint-enabled source caches are **not supported** for automated migration.

When presenting the migration plan to the user, if the source cache is VNet-injected, include Private Endpoint setup as a required pre-step before executing the automated migration.
```

- [ ] **Step 2: Update Step 3 (Plan Migration) — line 202**

Find this text:
```
3. **Network isolation**: ACR caches using VNet injection must be replaced with **Private Link** on AMR, as AMR does not support VNet injection. Ensure Private Endpoints are configured before cutover.
```

Replace with:
```
3. **Network isolation (VNET caches)**: If the source cache is VNet-injected, create a **Private Endpoint** on the target AMR cache before migration. AMR uses Private Link instead of VNet injection. Use the source cache's `subnetId` to identify the VNet — see [Private Endpoint Setup](references/private-endpoint-setup.md) for step-by-step CLI commands.
```

- [ ] **Step 3: Update Automated Migration exclusions — line 221**

Find this text:
```
- Supported: All Basic/Standard/Premium SKUs — **except** Private Link, VNet injected, or Geo-Replicated caches
```

Replace with:
```
- Supported: All Basic/Standard/Premium SKUs — **except** Private Link enabled or Geo-Replicated caches
- VNet-injected caches **are supported** but require Private Endpoint setup on the target AMR cache before migration — see [Private Endpoint Setup](references/private-endpoint-setup.md)
```

- [ ] **Step 4: Commit**

```bash
git add SKILL.md
git commit -m "Update SKILL.md with VNET detection guidance and PE setup flow"
```

---

### Task 3: Update `references/automated-migration.md`

**Files:**
- Modify: `references/automated-migration.md`

- [ ] **Step 1: Update the Supported Scope section (lines 17-22)**

Find this text:
```
**Supported source SKUs**: All Basic (C0–C6), Standard (C0–C6), and Premium (P1–P5) — **except**:
- Private Link enabled caches
- VNet injected caches
- Geo-Replication enabled caches

These exclusions are expected to be supported in future releases.
```

Replace with:
```
**Supported source SKUs**: All Basic (C0–C6), Standard (C0–C6), and Premium (P1–P5) — **except**:
- Private Link enabled caches
- Geo-Replication enabled caches

These exclusions are expected to be supported in future releases.

> **VNet-injected caches**: VNet-injected caches **are supported** for automated migration. However, since AMR does not support VNet injection, you must configure a **Private Endpoint** on the target AMR cache before migration to maintain network isolation. See [Private Endpoint Setup](private-endpoint-setup.md) for step-by-step instructions.
```

- [ ] **Step 2: Commit**

```bash
git add references/automated-migration.md
git commit -m "Remove VNET from automated migration exclusions, add PE pre-req"
```

---

### Task 4: Update `references/migration-validation.md`

**Files:**
- Modify: `references/migration-validation.md`

- [ ] **Step 1: Remove the VNET validation error (line 18)**

Find this text:
```
| Private endpoints on source | Remove private endpoints from the source before migration |
| VNet injection enabled on source | Use a non-VNet injected source cache |
```

Replace with:
```
| Private endpoints on source | Remove private endpoints from the source before migration |
```

- [ ] **Step 2: Commit**

```bash
git add references/migration-validation.md
git commit -m "Remove VNET injection validation error (no longer a blocker)"
```

---

### Task 5: Update `references/migration-overview.md`

**Files:**
- Modify: `references/migration-overview.md`

- [ ] **Step 1: Update the key highlights section (line 23)**

Find this text:
```
- AMR does **not** support scaling down/in or VNet injection
```

Replace with:
```
- AMR does **not** support scaling down/in or VNet injection (use [Private Link](private-endpoint-setup.md) instead for network isolation)
```

- [ ] **Step 2: Update the network isolation callout (line 43)**

Find this text:
```
> **Important — Network Isolation**: AMR does not support VNet injection. ACR Premium caches using VNet injection should be migrated to AMR with **Private Link** (Private Endpoints) for network isolation.
```

Replace with:
```
> **Important — Network Isolation**: AMR does not support VNet injection. ACR caches using VNet injection are supported for automated migration, but you must configure a **Private Endpoint** on the target AMR cache before migration. See [Private Endpoint Setup](private-endpoint-setup.md) for step-by-step CLI commands. The Private Endpoint should be created in the same VNet as the source cache's `subnetId`.
```

- [ ] **Step 3: Update the automated migration cons (line 109)**

Find this text:
```
**Cons**: Preview — limited scope (no Private Link, VNet, or geo-replicated caches); data migration not yet included
```

Replace with:
```
**Cons**: Preview — limited scope (no Private Link or geo-replicated source caches); VNet-injected caches require [Private Endpoint setup](private-endpoint-setup.md) on target; data migration not yet included
```

- [ ] **Step 4: Commit**

```bash
git add references/migration-overview.md
git commit -m "Update migration overview with VNET support and PE guidance"
```

---

### Task 6: Update `references/feature-comparison.md`

**Files:**
- Modify: `references/feature-comparison.md`

- [ ] **Step 1: Update the security table VNET row (line 74)**

Find this text:
```
| VNet Integration | ✅ (Premium) | ❌ (use Private Link) |
```

Replace with:
```
| VNet Integration | ✅ (Premium) | ❌ (use [Private Link](private-endpoint-setup.md) — VNet caches can still be migrated) |
```

- [ ] **Step 2: Update the Performance Tiers note (line 86)**

Find this text:
```
  - **Note**: AMR does not support VNet injection — use Private Link for network isolation instead
```

Replace with:
```
  - **Note**: AMR does not support VNet injection — use [Private Link](private-endpoint-setup.md) for network isolation instead. VNet-injected caches are supported for automated migration with Private Endpoint setup on the target.
```

- [ ] **Step 3: Commit**

```bash
git add references/feature-comparison.md
git commit -m "Update feature comparison with VNET migration guidance"
```

---

### Task 7: Update `references/azure-cli-commands.md`

**Files:**
- Modify: `references/azure-cli-commands.md`

- [ ] **Step 1: Add VNET Detection section after the Geo-Replication section (after line 76)**

Find this text:
```
If links are returned, the cache uses passive geo-replication (active geo-replication is an AMR feature, not available on ACR Premium).
```

After it, add:

```markdown

---

## VNET Detection (Premium)

Check if a cache uses VNet injection:

```bash
az redis show -g <resourceGroup> -n <cacheName> --query "{subnetId: subnetId}" -o json
```

If `subnetId` is not null, the cache is VNet-injected. Extract the VNet details:

```bash
# Get the full subnet ID
SUBNET_ID=$(az redis show -n <cacheName> -g <resourceGroup> --query "subnetId" -o tsv)

# Parse VNet and subnet names
VNET_RG=$(echo $SUBNET_ID | cut -d'/' -f5)
VNET_NAME=$(echo $SUBNET_ID | cut -d'/' -f9)
SUBNET_NAME=$(echo $SUBNET_ID | cut -d'/' -f11)

echo "VNet RG: $VNET_RG, VNet: $VNET_NAME, Subnet: $SUBNET_NAME"
```

These values are used when creating a Private Endpoint on the target AMR cache. See [Private Endpoint Setup](private-endpoint-setup.md).
```

- [ ] **Step 2: Update Recommended Fields to Collect (line 97)**

Find this text:
```
- **Network/TLS settings**
```

Replace with:
```
- **Network/TLS settings** (including `subnetId` for VNet-injected caches)
```

- [ ] **Step 3: Commit**

```bash
git add references/azure-cli-commands.md
git commit -m "Add VNET detection CLI commands"
```

---

### Task 8: Update Evals

**Files:**
- Modify: `evals/evals.json`

- [ ] **Step 1: Update eval ID 4 — VNET is no longer a blocker**

The current eval ID 4 prompt has a cache with VNET injection, private endpoints, AND geo-replication. Since VNET injection is no longer a blocker but private endpoints and geo-replication still are, update the eval to reflect this.

Find this text:
```json
    {
      "id": 4,
      "prompt": "I'm running a Premium P1 cache with VNet injection and private endpoints in eastus2. It also has geo-replication set up with a secondary in westus2. Can I use the automated migration API to move to AMR?",
      "expected_output": "Correctly identifies that this cache is NOT eligible for automated migration due to VNet injection, private endpoints, and geo-replication — all three are unsupported. Recommends manual migration strategies instead. Mentions the need to switch from VNet injection to Private Link on AMR.",
      "files": [],
      "expectations": [
        "Identifies VNet injection as a blocker for automated migration",
        "Identifies geo-replication as a blocker for automated migration",
        "Identifies private endpoints as a blocker for automated migration",
        "Suggests a manual migration strategy as an alternative",
        "Mentions that AMR uses Private Link instead of VNet injection"
      ]
    },
```

Replace with:
```json
    {
      "id": 4,
      "prompt": "I'm running a Premium P1 cache with private endpoints in eastus2. It also has geo-replication set up with a secondary in westus2. Can I use the automated migration API to move to AMR?",
      "expected_output": "Correctly identifies that this cache is NOT eligible for automated migration due to private endpoints and geo-replication — both are unsupported. Recommends manual migration strategies instead.",
      "files": [],
      "expectations": [
        "Identifies geo-replication as a blocker for automated migration",
        "Identifies private endpoints as a blocker for automated migration",
        "Suggests a manual migration strategy as an alternative"
      ]
    },
```

- [ ] **Step 2: Add new eval IDs 6 and 7 — VNET migration success path**

After the closing `}` of eval ID 5 (before the closing `]` of the evals array), add two new evals. The final section of the file should look like:

```json
    },
    {
      "id": 6,
      "prompt": "We have a Premium P2 cache with VNet injection in westus2. No private endpoints, no geo-replication. We want to migrate to AMR. Help us plan the migration including network setup.",
      "expected_output": "Detects that the cache is VNet-injected and explains that AMR does not support VNet injection. Recommends creating a Private Endpoint on the target AMR cache in the same VNet before migration. Provides or references CLI commands for Private Endpoint creation. Recommends an appropriate AMR SKU. Explains that automated migration is supported for VNet-injected caches. Mentions the port change (6380 to 10000).",
      "files": [],
      "expectations": [
        "Detects or mentions VNet injection on the source cache",
        "Explains that AMR uses Private Link instead of VNet injection",
        "Recommends creating a Private Endpoint on the target AMR cache before migration",
        "References the private-endpoint-setup guide or provides PE creation CLI commands",
        "States that automated migration IS supported for VNet-injected caches (not blocked)",
        "Recommends a specific AMR SKU (e.g., M20, M50) appropriate for a P2 cache",
        "Mentions the TLS port change from 6380 to 10000",
        "Mentions the Private DNS zone privatelink.redis.azure.net"
      ]
    },
    {
      "id": 7,
      "prompt": "I have a Premium P1 cache called 'vnet-cache' in resource group 'rg-prod'. It's VNet-injected. Can you look up its subnet info and tell me how to set up Private Link on my new AMR cache 'amr-cache' in the same RG?",
      "expected_output": "Uses az redis show to extract the subnetId from the source cache. Parses the VNet name, subnet name, and VNet resource group from the subnetId. Provides CLI commands to create a Private Endpoint on the AMR cache in the same VNet. Includes Private DNS zone setup with privatelink.redis.azure.net.",
      "files": [],
      "expectations": [
        "Runs or shows az redis show with a query for subnetId",
        "Extracts or explains how to parse VNet name and subnet from the subnetId path",
        "Provides az network private-endpoint create command with correct group-id (redisEnterprise)",
        "Includes Private DNS zone setup with privatelink.redis.azure.net",
        "Mentions the subnet delegation constraint (cannot reuse the delegated subnet for PE)"
      ]
    }
  ]
}
```

- [ ] **Step 3: Verify JSON is valid**

```bash
python3 -c "import json; json.load(open('evals/evals.json')); print('JSON valid')"
```

Expected: `JSON valid`

- [ ] **Step 4: Commit**

```bash
git add evals/evals.json
git commit -m "Update evals: remove VNET blocker, add VNET migration success path tests"
```
