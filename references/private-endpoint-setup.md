# Private Endpoint Setup for AMR (VNET Migration)

When migrating a VNet-injected ACR cache to AMR, you must set up a **Private Endpoint** on the target AMR cache to maintain network isolation. AMR does not support VNet injection — Private Link (via Private Endpoints) is the replacement.

---

## Step 1: Extract VNet Info from the Source Cache

Get the subnet ID from the source ACR cache to identify which VNet/subnet to create the Private Endpoint in:

```bash
az redis show -n <cacheName> -g <resourceGroup> --query "subnetId" -o tsv
```

This returns a full ARM resource ID like:
```
/subscriptions/<subId>/resourceGroups/<rg>/providers/Microsoft.Network/virtualNetworks/<vnetName>/subnets/<subnetName>
```

Extract the VNet name and subnet name from this path — you'll use the same VNet (and optionally the same or a different subnet) for the Private Endpoint.

To also get the VNet's resource group (which may differ from the cache's resource group):

```bash
# Parse the VNet resource group from the subnetId
SUBNET_ID=$(az redis show -n <cacheName> -g <resourceGroup> --query "subnetId" -o tsv)
VNET_RG=$(echo $SUBNET_ID | cut -d'/' -f5)
VNET_NAME=$(echo $SUBNET_ID | cut -d'/' -f9)
SUBNET_NAME=$(echo $SUBNET_ID | cut -d'/' -f11)
echo "VNet RG: $VNET_RG, VNet: $VNET_NAME, Subnet: $SUBNET_NAME"
```

---

## Step 2: Create a Private Endpoint on the AMR Cache

Create a Private Endpoint in the same VNet as the source cache. Use a subnet that does **not** have the `Microsoft.Cache/Redis` service delegation (VNET-injected caches use a delegated subnet that cannot be shared with Private Endpoints).

> **Subnet note**: The subnet used for VNet injection has a service delegation to `Microsoft.Cache/Redis`. You can use a **different subnet** in the same VNet for the Private Endpoint. If needed, create a new subnet:
>
> ```bash
> az network vnet subnet create \
>   --resource-group <vnetResourceGroup> \
>   --vnet-name <vnetName> \
>   --name pe-subnet \
>   --address-prefix 10.0.2.0/24
> ```

Create the Private Endpoint:

```bash
az network private-endpoint create \
  --name <amrCacheName>-pe \
  --resource-group <resourceGroup> \
  --vnet-name <vnetName> \
  --subnet <subnetName> \
  --private-connection-resource-id /subscriptions/<subId>/resourceGroups/<rg>/providers/Microsoft.Cache/redisEnterprise/<amrCacheName> \
  --group-id redisEnterprise \
  --connection-name <amrCacheName>-connection
```

> **Parameters**:
> - `--private-connection-resource-id`: Full ARM resource ID of the **target AMR cache** (uses `Microsoft.Cache/redisEnterprise` provider)
> - `--group-id`: Must be `redisEnterprise` for AMR caches
> - `--subnet`: Use a subnet without the `Microsoft.Cache/Redis` delegation

---

## Step 3: Configure Private DNS Zone

Create a Private DNS zone and link it to the VNet so the AMR cache hostname resolves to the Private Endpoint's private IP:

```bash
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
```

> **DNS zone**: AMR uses `privatelink.redis.azure.net`. This is different from OSS ACR (`privatelink.redis.cache.windows.net`) and legacy Enterprise (`privatelink.redisenterprise.cache.azure.net`).

---

## Step 4: Validate Connectivity

Verify the Private Endpoint is working before proceeding with migration:

```bash
# Check Private Endpoint status
az network private-endpoint show \
  --name <amrCacheName>-pe \
  --resource-group <resourceGroup> \
  --query "privateLinkServiceConnections[0].privateLinkServiceConnectionState.status" -o tsv
```

Expected output: `Approved`

To verify DNS resolution from within the VNet (run from a VM in the same VNet):

```bash
nslookup <amrCacheName>.<region>.redis.azure.net
```

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
