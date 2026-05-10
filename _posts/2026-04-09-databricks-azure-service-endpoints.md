---
layout: post
title: "Cut Your Azure Bill: Use Service Endpoints for Databricks Storage Access"
date: 2026-04-09
categories: [azure, databricks, cost-optimization]
tags: [azure, databricks, adls, service-endpoints, networking, cost]
---

Every byte of data your Databricks clusters read from Azure Storage travels a path — and that path determines whether you pay for it. If your workloads are moving significant volumes of data and you haven't configured **Azure Service Endpoints** (or Private Endpoints), you may be paying for egress traffic you could eliminate entirely.

This post breaks down why the default routing is expensive, what service endpoints do differently, and exactly how to set them up.

---

## The Default Path and Why It Costs Money

Without service endpoints, traffic from your Databricks clusters to Azure Data Lake Storage (ADLS Gen2) or Blob Storage takes a detour through the public internet — even if both resources live in the same Azure region.

The flow looks like this:

```
Databricks Worker Node
  → Azure VNet
    → Public Internet (egress charge applied)
      → Azure Storage public endpoint
```

Azure charges for **outbound data transfer** leaving a VNet to a public endpoint. At scale — think terabytes of Parquet reads, Delta Lake merges, or ML training data — this adds up fast.

> **Example cost:** 50 TB/month of storage reads at $0.087/GB (standard Azure egress) = **~$4,350/month in avoidable charges.**

---

## What Service Endpoints Do

A Service Endpoint creates a direct, optimized route from your VNet subnet to the Azure Storage service over the **Azure backbone network** — bypassing the public internet entirely.

The new flow:

```
Databricks Worker Node
  → Azure VNet (with Service Endpoint enabled)
    → Azure backbone (no public internet, no egress charge)
      → Azure Storage
```

Key benefits:

- **Egress charges eliminated** for storage traffic within the same region
- **Lower latency** — backbone routing is faster than public internet paths
- **Improved security** — storage can be locked down to only accept traffic from your VNet subnet
- **No additional cost** — service endpoints themselves are free

---

## Service Endpoints vs. Private Endpoints

Both solve the same routing problem, but they differ in complexity and security posture:

| | Service Endpoint | Private Endpoint |
|---|---|---|
| **Cost** | Free | ~$7.30/month per endpoint + data charges |
| **Setup complexity** | Low | Medium–High |
| **Network path** | VNet → Azure backbone | Private IP in your VNet |
| **Storage firewall** | VNet-scoped | IP-scoped |
| **DNS changes required** | No | Yes |
| **Best for** | Most Databricks workloads | Highly regulated environments |

For most teams, **Service Endpoints are the right starting point** — free, fast to configure, and they eliminate the egress cost entirely.

---

## Step-by-Step: Enabling Service Endpoints for Databricks

### Prerequisites

- An existing Databricks workspace deployed in your own VNet (VNet injection)
- Contributor access to the VNet and Storage Account
- Your Databricks subnets: typically a **public** and **private** subnet

> **Note:** If your Databricks workspace uses the default managed VNet, you'll need to migrate to a customer-managed VNet (VNet injection) first to control subnet settings.

---

### Step 1: Enable the Service Endpoint on Your Subnets

In the Azure Portal, navigate to your VNet → **Subnets**, and update both the Databricks public and private subnets.

For each subnet, under **Service endpoints**, add:

```
Microsoft.Storage
```

Or via Azure CLI:

```bash
# Enable on the private subnet
az network vnet subnet update \
  --resource-group <your-rg> \
  --vnet-name <your-vnet> \
  --name <databricks-private-subnet> \
  --service-endpoints Microsoft.Storage

# Enable on the public subnet
az network vnet subnet update \
  --resource-group <your-rg> \
  --vnet-name <your-vnet> \
  --name <databricks-public-subnet> \
  --service-endpoints Microsoft.Storage
```

---

### Step 2: Lock Down the Storage Account Firewall

Once the endpoint is active, restrict the storage account to only accept traffic from your VNet subnets. This is optional but strongly recommended.

In the Azure Portal → your Storage Account → **Networking** → **Firewalls and virtual networks**:

1. Set **"Allow access from"** to **"Selected networks"**
2. Under **Virtual networks**, click **+ Add existing virtual network**
3. Select your VNet and both Databricks subnets
4. Click **Save**

Via CLI:

```bash
az storage account network-rule add \
  --resource-group <your-rg> \
  --account-name <your-storage-account> \
  --vnet-name <your-vnet> \
  --subnet <databricks-private-subnet>

az storage account network-rule add \
  --resource-group <your-rg> \
  --account-name <your-storage-account> \
  --vnet-name <your-vnet> \
  --subnet <databricks-public-subnet>

az storage account update \
  --resource-group <your-rg> \
  --name <your-storage-account> \
  --default-action Deny
```

> **Important:** If you deny all access and forget to add your admin IP or another allowed network, you'll lock yourself out. Always add your own IP to the allowlist during testing.

---

### Step 3: Verify Databricks Can Still Read Storage

Mount or access your storage from a Databricks notebook and confirm reads work:

```python
# Test read from ADLS Gen2 with OAuth / service principal
spark.conf.set(
    "fs.azure.account.auth.type.<your-storage-account>.dfs.core.windows.net",
    "OAuth"
)
spark.conf.set(
    "fs.azure.account.oauth.provider.type.<your-storage-account>.dfs.core.windows.net",
    "org.apache.hadoop.fs.azurebfs.oauth2.ClientCredsTokenProvider"
)
spark.conf.set(
    "fs.azure.account.oauth2.client.id.<your-storage-account>.dfs.core.windows.net",
    "<your-client-id>"
)
spark.conf.set(
    "fs.azure.account.oauth2.client.secret.<your-storage-account>.dfs.core.windows.net",
    dbutils.secrets.get(scope="<scope>", key="<secret-key>")
)
spark.conf.set(
    "fs.azure.account.oauth2.client.endpoint.<your-storage-account>.dfs.core.windows.net",
    "https://login.microsoftonline.com/<your-tenant-id>/oauth2/token"
)

# Quick validation
df = spark.read.format("delta").load(
    "abfss://<container>@<your-storage-account>.dfs.core.windows.net/<path>"
)
df.printSchema()
print(f"Row count: {df.count()}")
```

If the read succeeds, traffic is now flowing via the service endpoint.

---

### Step 4: Verify Routing in Azure (Optional)

You can confirm the effective routes on a NIC attached to a VM in your Databricks subnet using Azure Network Watcher → **Effective Routes**. You should see a route for the storage prefix pointing to **VirtualNetworkServiceEndpoint** rather than the internet.

---

## Common Gotchas

**Databricks in the default managed VNet**
Service endpoints require subnet-level control. If you're not using VNet injection, you can't configure this. Migrating to VNet injection is the prerequisite — it's worth doing for security and cost control in production environments.

**Multiple storage accounts**
You need to add the VNet rule to each storage account individually. Automate this with a script or Azure Policy if you have many accounts.

**Azure Data Factory or other services also reading the storage**
If you set the storage firewall to "Selected networks only," any other service (ADF, Azure ML, etc.) that previously accessed storage over the public internet will also need to be added to the allowlist or configured with managed private endpoints.

**Databricks cluster logs and DBFS root**
The DBFS root storage and cluster log delivery paths are managed by Databricks, not by your VNet. Those won't be affected. Focus on your own data storage accounts.

---

## Estimating Your Savings

Use this rough formula:

```
Monthly egress savings = (TB read/month) × 1024 × $0.087
```

| Monthly Read Volume | Estimated Monthly Savings |
|---|---|
| 5 TB | ~$446 |
| 20 TB | ~$1,782 |
| 50 TB | ~$4,454 |
| 100 TB | ~$8,909 |

At enterprise scale, this is one of the highest-ROI Azure cost optimizations available — free to implement, permanent in effect.

---

## Wrapping Up

Service endpoints are one of the simplest and most impactful changes you can make to a Databricks-on-Azure architecture. If your clusters are doing significant storage I/O, the egress charges you're currently paying are unnecessary. The setup takes under 30 minutes and the savings are immediate.

Next steps:
- Enable `Microsoft.Storage` service endpoints on your Databricks subnets
- Restrict your storage accounts to only your VNet
- Validate with a test read from a cluster notebook
- Check your Azure Cost Management dashboard the next billing cycle

Have questions or a different setup (like Private Endpoints or Unity Catalog managed storage)? Reach out at [info@databucks.ai](mailto:info@databucks.ai).

---

*Posted by the databucks.ai team · [Back to home](/) · [info@databucks.ai](mailto:info@databucks.ai)*
