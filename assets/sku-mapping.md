# SKU Mapping Guide: Azure Cache for Redis to Azure Managed Redis

This guide helps you select the appropriate Azure Managed Redis (AMR) SKU when migrating from Azure Cache for Redis.

> **Source**: Internal SKU mapping spreadsheet (ACR_AMR.xlsx)
> 
> **Last Updated**: February 2026

---

## 💰 Dynamic Pricing

Use the pricing scripts to get real-time pricing with monthly cost calculations:

```powershell
# Windows PowerShell - AMR pricing (HA by default)
.\scripts\get_redis_price.ps1 -SKU M10 -Region westus2
.\scripts\get_redis_price.ps1 -SKU M10 -Region westus2 -NoHA

# Windows PowerShell - ACR pricing (specify tier for C* SKUs)
.\scripts\get_redis_price.ps1 -SKU C3 -Region westus2 -Tier Standard
.\scripts\get_redis_price.ps1 -SKU P2 -Region westus2
.\scripts\get_redis_price.ps1 -SKU P2 -Region westus2 -Shards 3 -Replicas 2

# Linux/Mac
./scripts/get_redis_price.sh M10 westus2
./scripts/get_redis_price.sh P2 westus2 --shards 3
```

The scripts automatically calculate monthly costs based on:
- **AMR**: HA (×2 nodes) or non-HA (×1 node)
- **ACR Basic**: Single node (×1)
- **ACR Standard**: Always HA (×2 nodes)
- **ACR Premium**: HA × shards × replicas per shard

See [pricing-tiers.md](../references/pricing-tiers.md) for detailed calculation rules.

> **Official Quotes**: Use [Azure Pricing Calculator](https://azure.microsoft.com/en-us/pricing/calculator/?service=managed-redis)

---

## ⚠️ Important: Memory Reservation

**Both Azure Cache for Redis (ACR) and Azure Managed Redis (AMR) reserve ~20% of memory for system overhead.**

| Metric | Description |
|--------|-------------|
| **Advertised Size** | The SKU label (e.g., M20 = 24 GB, B50 = 60 GB) |
| **Usable Memory** | ~80% of advertised size available for your data |

### Example Calculation
- **M50 SKU**: Advertised 60 GB → **~48 GB usable** for data
- **B100 SKU**: Advertised 120 GB → **~96 GB usable** for data

### Migration Sizing Rule
When migrating, compare **usable memory to usable memory**:
1. Check your current ACR cache's **actual used memory** (not SKU size) in Azure Portal
2. Select an AMR SKU where **80% of the advertised size** exceeds your peak usage
3. Add 20-30% buffer for growth

**Example**: If your P2 cache (13 GB advertised, ~10.4 GB usable) is using 8 GB of data:
- Need AMR with at least 8 GB usable + growth buffer
- M10 (12 GB × 80% = 9.6 GB usable) - tight fit, no growth room
- **M20 (24 GB × 80% = 19.2 GB usable) - recommended** ✓

---

## Mapping Guiding Principles

1. Choose the lowest memory configuration with the least number of shards that maps to your existing ACR configuration
2. Memory Optimized (M-series) is the cheapest option but starts from M10
3. For clustered caches, Balanced (B-series) provides more shards at comparable cost to Premium
4. For compute intensive tasks Compute Optimized (X-Series) offer more compute power for the same amount of memory, but at a higher cost.
5. **Always calculate usable memory (advertised × 0.8) when comparing SKUs**

---

## AMR SKU Definitions

> **Note**: "Advertised" is the SKU size. Usable memory is ~80% of this value.

### Memory Optimized (M-Series)
Best for: Caching workloads with large datasets and moderate throughput requirements. Memory-to-vCPU ratio: 8:1.

| SKU | Advertised (GB) | Usable ~(GB) | vCPUs | Max Connections |
|-----|-----------------|--------------|-------|-----------------|
| M10 | 12 | 9.6 | 2 | 15,000 |
| M20 | 24 | 19.2 | 4 | 30,000 |
| M50 | 60 | 48 | 8 | 75,000 |
| M100 | 120 | 96 | 16 | 150,000 |
| M150 | 175 | 140 | 24 | 200,000 |
| M250 | 235 | 188 | 32 | 200,000 |
| M350* | 360 | 288 | 48 | 200,000 |
| M500* | 480 | 384 | 64 | 200,000 |
| M700* | 720 | 576 | 96 | 200,000 |
| M1000* | 960 | 768 | 128 | 200,000 |
| M1500* | 1440 | 1152 | 192 | 200,000 |
| M2000* | 1920 | 1536 | 256 | 200,000 |

### Balanced (B-Series)
Best for: General-purpose workloads with balanced compute and memory needs. Memory-to-vCPU ratio: 4:1.

| SKU | Advertised (GB) | Usable ~(GB) | vCPUs | Max Connections |
|-----|-----------------|--------------|-------|-----------------|
| B0 | 0.5 | 0.4 | 2 (burstable) | 15,000 |
| B1 | 1 | 0.8 | 2 (burstable) | 15,000 |
| B3 | 3 | 2.4 | 2 | 15,000 |
| B5 | 6 | 4.8 | 2 | 15,000 |
| B10 | 12 | 9.6 | 4 | 30,000 |
| B20 | 24 | 19.2 | 8 | 75,000 |
| B50 | 60 | 48 | 16 | 150,000 |
| B100 | 120 | 96 | 32 | 200,000 |
| B150 | 180 | 144 | 48 | 200,000 |
| B250 | 240 | 192 | 64 | 200,000 |
| B350* | 360 | 288 | 96 | 200,000 |
| B500* | 480 | 384 | 128 | 200,000 |
| B700* | 720 | 576 | 192 | 200,000 |
| B1000* | 960 | 768 | 256 | 200,000 |

### Compute Optimized (X-Series)
Best for: Performance-intensive workloads requiring maximum throughput. Memory-to-vCPU ratio: 2:1.

| SKU | Advertised (GB) | Usable ~(GB) | vCPUs | Max Connections |
|-----|-----------------|--------------|-------|-----------------|
| X3 | 3 | 2.4 | 4 | 30,000 |
| X5 | 6 | 4.8 | 4 | 30,000 |
| X10 | 12 | 9.6 | 8 | 75,000 |
| X20 | 24 | 19.2 | 16 | 150,000 |
| X50 | 60 | 48 | 32 | 200,000 |
| X100 | 120 | 96 | 64 | 200,000 |
| X150 | 180 | 144 | 96 | 200,000 |
| X250 | 240 | 192 | 128 | 200,000 |
| X350* | 360 | 288 | 192 | 200,000 |
| X500* | 480 | 384 | 256 | 200,000 |
| X700* | 720 | 576 | 384 | 200,000 |

\* Sizes marked with asterisk are in Public Preview.

### Flash Optimized (Auto Tiering)
Best for: Very large datasets where tiered storage (RAM + SSD) is cost-effective.

> **Note**: Flash Optimized uses a combination of RAM and NVMe disk. The "Cache Size" represents total effective capacity. Memory reservation applies to the RAM portion. More info: [Flash Optimized Architecture](https://learn.microsoft.com/en-us/azure/redis/architecture#flash-optimized-tier)

| SKU | Cache Size (GB) | VM Memory (GiB) | Disk Size (GiB) | vCPUs | Max Connections |
|-----|-----------------|-----------------|-----------------|-------|-----------------|
| A250* | 250 | 64 | 1920 | 8 | 75,000 |
| A500* | 500 | 128 | 3840 | 16 | 150,000 |
| A700* | 750 | 192 | 5760 | 24 | 200,000 |
| A1000* | 1000 | 256 | 7680 | 32 | 200,000 |
| A1500* | 1500 | 384 | 11520 | 48 | 200,000 |
| A2000* | 2000 | 512 | 15360 | 64 | 200,000 |
| A4500* | 4500 | 1152 | 34560 | 144 | 200,000 |

\* All Flash Optimized SKUs are in Public Preview.

---

## Basic/Standard Non-Clustered → AMR Mapping

| Tier | SKU | ACR Usable | Target AMR | AMR Advertised (Usable) |
|------|-----|------------|------------|-------------------------|
| **Basic** | C0 | 0.2 GB | B0 | 0.5 GB (0.4 GB) |
| | C1 | 0.8 GB | B1 | 1 GB (0.8 GB) |
| | C2 | 2 GB | B3 | 3 GB (2.4 GB) |
| | C3 | 4.8 GB | B5 | 6 GB (4.8 GB) |
| | C4 | 10.4 GB | M10 or M20 | 12 GB (9.6 GB) or 24 GB (19.2 GB) |
| | C5 | 20.8 GB | M20 or M50 | 24 GB (19.2 GB) or 60 GB (48 GB) |
| | C6 | 42.4 GB | M50 | 60 GB (48 GB) |
| **Standard** | C0 | 0.2 GB | B0 | 0.5 GB (0.4 GB) |
| | C1 | 0.8 GB | B1 | 1 GB (0.8 GB) |
| | C2 | 2 GB | B3 | 3 GB (2.4 GB) |
| | C3 | 4.8 GB | B5 | 6 GB (4.8 GB) |
| | C4 | 10.4 GB | M10 or M20 | 12 GB (9.6 GB) or 24 GB (19.2 GB) |
| | C5 | 20.8 GB | M20 or M50 | 24 GB (19.2 GB) or 60 GB (48 GB) |
| | C6 | 42.4 GB | M50 | 60 GB (48 GB) |

**Note**: Use the smaller SKU (M10, M20) for cost efficiency if your peak memory usage fits. Use larger SKU (M20, M50) for growth headroom.

---

## Premium Non-Clustered → AMR Mapping

| SKU | ACR Usable | Target AMR | AMR Advertised (Usable) |
|-----|------------|------------|-------------------------|
| P1 | 4.8 GB | B5 | 6 GB (4.8 GB) |
| P2 | 10.4 GB | B10 or B20 | 12 GB (9.6 GB) or 24 GB (19.2 GB) |
| P3 | 20.8 GB | B20 or B50 | 24 GB (19.2 GB) or 60 GB (48 GB) |
| P4 | 42.4 GB | B50 | 60 GB (48 GB) |
| P5 | 96 GB | B100 | 120 GB (96 GB) |

---

## Premium Clustered → AMR Mapping

### P1 Clustered (6 GB per shard)

| Shards | ACR Usable | Target AMR | AMR Advertised (Usable) |
|--------|------------|------------|-------------------------|
| 1 | 4.8 GB | B5 | 6 GB (4.8 GB) |
| 2 | 9.6 GB | B10 | 12 GB (9.6 GB) |
| 3 | 14.4 GB | B20 | 24 GB (19.2 GB) |
| 4 | 19.2 GB | B20 | 24 GB (19.2 GB) |
| 5 | 24 GB | B50 | 60 GB (48 GB) |
| 6 | 28.8 GB | B50 | 60 GB (48 GB) |
| 7 | 33.6 GB | B50 | 60 GB (48 GB) |
| 8 | 38.4 GB | B50 | 60 GB (48 GB) |
| 9 | 43.2 GB | B50 | 60 GB (48 GB) |
| 10 | 48 GB | B50 | 60 GB (48 GB) |
| 11-15 | 52.8-72 GB | B100 | 120 GB (96 GB) |

### P2 Clustered (13 GB per shard)

| Shards | ACR Usable | Target AMR | AMR Advertised (Usable) |
|--------|------------|------------|-------------------------|
| 1 | 10.4 GB | B20 | 24 GB (19.2 GB) |
| 2 | 20.8 GB | B50 | 60 GB (48 GB) |
| 3 | 31.2 GB | B50 | 60 GB (48 GB) |
| 4 | 41.6 GB | B50 | 60 GB (48 GB) |
| 5 | 52 GB | B100 | 120 GB (96 GB) |
| 6 | 62.4 GB | B100 | 120 GB (96 GB) |
| 7 | 72.8 GB | B100 | 120 GB (96 GB) |
| 8 | 83.2 GB | B100 | 120 GB (96 GB) |
| 9 | 93.6 GB | B100 | 120 GB (96 GB) |
| 10 | 104 GB | B150 | 180 GB (144 GB) |
| 11-12 | 114-125 GB | B150 | 180 GB (144 GB) |
| 13-14 | 135-146 GB | B250 | 240 GB (192 GB) |
| 15 | 156 GB | B250 | 240 GB (192 GB) |

### P3 Clustered (26 GB per shard)

| Shards | ACR Usable | Target AMR | AMR Advertised (Usable) |
|--------|------------|------------|-------------------------|
| 1 | 20.8 GB | B50 | 60 GB (48 GB) |
| 2 | 41.6 GB | B50 | 60 GB (48 GB) |
| 3 | 62.4 GB | B100 | 120 GB (96 GB) |
| 4 | 83.2 GB | B100 | 120 GB (96 GB) |
| 5 | 104 GB | B150 | 180 GB (144 GB) |
| 6 | 124.8 GB | B150 | 180 GB (144 GB) |
| 7 | 145.6 GB | B250 | 240 GB (192 GB) |
| 8 | 166.4 GB | B250 | 240 GB (192 GB) |
| 9 | 187.2 GB | B250 | 240 GB (192 GB) |
| 10 | 208 GB | B350 | 360 GB (288 GB) |
| 11-12 | 229-250 GB | B350 | 360 GB (288 GB) |
| 13 | 270.4 GB | B350 | 360 GB (288 GB) |
| 14-15 | 291-312 GB | B500 | 480 GB (384 GB) |

### P4 Clustered (53 GB per shard)

| Shards | ACR Usable | Target AMR | AMR Advertised (Usable) |
|--------|------------|------------|-------------------------|
| 1 | 42.4 GB | B50 | 60 GB (48 GB) |
| 2 | 84.8 GB | B100 | 120 GB (96 GB) |
| 3 | 127.2 GB | B150 | 180 GB (144 GB) |
| 4 | 169.6 GB | B250 | 240 GB (192 GB) |
| 5 | 212 GB | B350 | 360 GB (288 GB) |
| 6 | 254.4 GB | B350 | 360 GB (288 GB) |
| 7 | 296.8 GB | B500 | 480 GB (384 GB) |
| 8 | 339.2 GB | B500 | 480 GB (384 GB) |
| 9 | 381.6 GB | B500 | 480 GB (384 GB) |
| 10 | 424 GB | B700 | 720 GB (576 GB) |
| 11-13 | 466-551 GB | B700 | 720 GB (576 GB) |
| 14-15 | 594-636 GB | B1000 | 960 GB (768 GB) |

### P5 Clustered (120 GB per shard)

| Shards | ACR Usable | Target AMR | AMR Advertised (Usable) |
|--------|------------|------------|-------------------------|
| 1 | 96 GB | B100 | 120 GB (96 GB) |
| 2 | 192 GB | B250 | 240 GB (192 GB) |
| 3 | 288 GB | B350 | 360 GB (288 GB) |
| 4 | 384 GB | B350 | 360 GB (288 GB) |
| 5 | 480 GB | B700 | 720 GB (576 GB) |
| 6 | 576 GB | B700 | 720 GB (576 GB) |
| 7 | 672 GB | B1000 | 960 GB (768 GB) |
| 8 | 768 GB | B1000 | 960 GB (768 GB) |
| 9 | 864 GB | M1500 | 1440 GB (1152 GB) |
| 10-12 | 960-1152 GB | M1500 | 1440 GB (1152 GB) |
| 13-15 | 1248-1440 GB | M2000 | 1920 GB (1536 GB) |

---

## Selection Criteria

When selecting an AMR SKU, consider:

1. **Memory Requirements**
   - Current **actual used memory** (not SKU size) from Azure Portal metrics
   - Both ACR and AMR reserve ~20% for system overhead
   - Add 20-30% buffer for growth projections

2. **Throughput Requirements**
   - Current operations/second
   - Peak vs average load
   - Read/write ratio

3. **Feature Requirements**
   - Clustering needs (B-series provides more shards)
   - Geo-replication requirements
   - Redis module usage (JSON, Search, etc.)

4. **Cost Considerations**
   - M-series is cheapest but has fewer shards
   - B-series recommended for clustered caches
   - Non-HA options available for dev/test (50% savings)

---

## Choosing the Right Performance Tier

### Tier Comparison

| Tier | Memory:vCPU Ratio | Best For |
|------|-------------------|----------|
| **Memory Optimized (M)** | 8:1 | Memory-intensive workloads, large datasets, lower throughput needs, dev/test |
| **Balanced (B)** | 4:1 | Standard workloads, good balance of memory and compute |
| **Compute Optimized (X)** | 2:1 | High-throughput, low-latency, performance-intensive workloads |
| **Flash Optimized** | RAM + NVMe | Very large datasets, cost-effective scaling with tiered storage |

### When to Choose Compute Optimized (X-Series)

Choose Compute Optimized when your existing cache has:

1. **Low memory utilization but high operations/second**
   - Your cache is using < 50% of available memory
   - But you're hitting CPU limits or experiencing latency issues
   - Example: C3 Standard (6 GB) using only 2 GB memory but running 50K+ ops/sec

2. **Real-time applications requiring lowest latency**
   - Gaming leaderboards
   - Real-time analytics dashboards
   - Session management with high concurrent users

3. **High connection counts**
   - Compute Optimized SKUs support more max connections at each size
   - Example: X10 (12 GB) supports 75,000 connections vs M10's 15,000

4. **Workloads with complex Redis commands**
   - Heavy use of Lua scripts
   - Complex sorted set operations
   - Search/query operations with RediSearch

### Max Connections by Tier (at same memory size)

| Size (GB) | Memory Optimized | Balanced | Compute Optimized |
|-----------|------------------|----------|-------------------|
| 12 | 15,000 | 30,000 | 75,000 |
| 24 | 30,000 | 75,000 | 150,000 |
| 60 | 75,000 | 150,000 | 200,000 |
| 120 | 150,000 | 200,000 | 200,000 |

### Migration Decision Matrix

| Current Situation | Recommended AMR Tier |
|-------------------|---------------------|
| Memory usage > 70%, moderate ops | **Memory Optimized (M)** |
| Memory usage 40-70%, balanced workload | **Balanced (B)** |
| Memory usage < 40%, high ops/sec | **Compute Optimized (X)** |
| Very large dataset, cost-sensitive | **Flash Optimized** |
| Unsure / General purpose | **Balanced (B)** - start here |

### How to Assess Your Current Cache

Before migrating, check these metrics in Azure Portal for your existing cache:

1. **Memory Usage**: Monitor → Used Memory / Max Memory
   - If consistently < 50%, consider smaller SKU or Compute Optimized
   
2. **Operations per Second**: Monitor → Operations Per Second
   - High ops (>50K/sec) with low memory = Compute Optimized candidate
   
3. **Server Load**: Monitor → Server Load
   - High Server Load (>70%) with available memory = Compute Optimized candidate
   
4. **Connected Clients**: Monitor → Connected Clients
   - Approaching max connections = move to higher tier or Compute Optimized

### Example Migration Scenarios

**Scenario 1: Session Store with High Concurrency**
- Current: Premium P2 (13 GB), using 4 GB memory, 80K ops/sec, 25K connections
- Recommendation: **X10 (12 GB)** - provides 75K connections and high throughput

**Scenario 2: Application Cache with Large Dataset**
- Current: Premium P3 (26 GB), using 22 GB memory, 20K ops/sec
- Recommendation: **M20 or B20 (24 GB)** - memory-focused, adequate throughput

**Scenario 3: Real-time Analytics Dashboard**
- Current: Premium P1 (6 GB), using 2 GB memory, 100K ops/sec, Server Load at 85%
- Recommendation: **X5 (6 GB)** - maximum throughput for compute-intensive workload

**Scenario 4: General Web App Cache**
- Current: Standard C3 (6 GB), using 4 GB memory, 15K ops/sec
- Recommendation: **B5 (6 GB)** - balanced option, room to grow
