# OpenStack Freezer Billing and Metrics Guide

## Overview

This document provides a comprehensive pricing analysis for OpenStack Freezer (Backup-as-a-Service) compared to public cloud equivalents, along with detailed use cases and Ceilometer metrics to forward to Atom-Hopper/UMS billing implementation.

---

## How Does Freezer Work?

### High-Level Product Behavior (5-Minute Overview)

**Freezer** is OpenStack's Backup-as-a-Service (BaaS) component that provides automated backup, restore, and disaster recovery capabilities for various OpenStack resources and external systems.

#### Core Functionality

1. **Backup Agent Architecture**
   - Freezer agents run on dedicated backup nodes/VM's
   - Agents communicate with Freezer API(running on flex) to receive backup jobs
   - Support for multiple backup engines (rsync, tar, OpenStack native APIs)

   ![https://docs.openstack.org/freezer/latest/_images/Service_Architecture_02.png](https://docs.openstack.org/freezer/latest/_images/Service_Architecture_02.png)

2. **High Level Backup Workflow**
   ```
   User → Freezer Agent → Freezer API → Storage Backend
              ↓                           (Swift/S3)
          Source Data 
        (VM/DB/Volume)
   ```

3. **Supported Backup Types**
   - **File System Backups**: 
     - Local filesystems/directories
     - LVM
   - **Database Backups**: 
     - MySQL
     - MongoDB
   - **OpenStack Resource Backups**: 
     - Nova VMs (QCOW2 image based, RAW image based)
     - Cinder volumes (incremental block-level)

4. **Key Features**
   - **Incremental Backups**: Only changed blocks/files are backed up
   - **Compression**: zlib, bzip2, xz
   - **Encryption**: AES-256-CFB encryption for data at rest and in transit
   - **Deduplication**: Block-level and file-level deduplication
   - **Scheduling**: Cron-like scheduling for automated backups
   - **Multi-tenancy**: Project-isolated backup namespaces

5. **Storage Backend Support**
   - Swift (OpenStack Object Storage)
   - S3-compatible storage **(TBD)**
   - Local filesystem
   - SSH/SFTP remote storage

6. **Restore Process**
   - Point-in-time recovery from any backup/snapshot (timestamp based)
   - Selective file/directory restore
   - Full VM or Volume restoration

#### Typical Backup Lifecycle

```
1. Job Creation → User defines backup job via API/CLI
2. Scheduling → Freezer scheduler triggers job at specified time
3. Pre-Backup → Agent prepares source (quiesce DB - needs support via shell script, snapshot VM)
4. Backup Execution → Data is read, compressed, encrypted
5. Transfer → Backup data sent to storage backend
6. Metadata Update → Freezer API records backup metadata (stores in mariadb)
7. Post-Backup → notification (logs)
8. Retention Management → Old can be backups pruned (a policy mechanism can be devised as a supporting mechanism)
```

---

## Resource Consumption

### What Resources Does Freezer Consume?

#### 1. **Compute Resources**

| Resource Type | Consumption Pattern | Typical Usage |
|---------------|---------------------|---------------|
| **CPU** | Burst during backup/restore | 1-4 cores per agent during active backup |
| **Memory** | Proportional to backup buffer size | 512MB - 4GB per agent |
| **Network Bandwidth** | Sustained during transfer | 100Mbps - 10Gbps depending on backup size |
| **Disk I/O** | High during source read | Depends on source data size and type |

#### 2. **Storage Resources**

| Storage Type | Purpose | Growth Rate |
|--------------|---------|-------------|
| **Backup Storage** | Actual backup data | Linear with data size × retention period |
| **Metadata Storage** | Job definitions, backup catalogs | Minimal (~1MB per 1000 backups) |
| **Temporary Storage** | Staging area during backup | Equal to largest single backup job |
| **Database Storage** | Freezer API database (MariaDB) | ~100MB - 1GB for metadata |

#### 3. **Network Resources**

| Traffic Type | Direction | Volume |
|--------------|-----------|--------|
| **Backup Operation** | Freezer Agent → Storage Backend | Full backup size (compressed) |
| **Restore Operation** | Storage Backend → Agent | Restored data size |
| **API Communication** | Client/Agent ↔ Freezer API | Minimal (~1KB per API call) |
| **Metadata Sync** | Freezer API ↔ Database(MariaDB) | ~10KB per backup job |

#### 4. **OpenStack Service Dependencies**

- **Keystone**: Authentication and authorization
- **Swift/Cinder**: Storage backend (if used)
- **Nova**: VM snapshot operations
- **Glance**: Image operations
- **Cinder**: Volume snapshot operations
- **RabbitMQ**: Message queue for job distribution
- **MariaDB**: Metadata and job state storage

---

## Ceilometer Metrics Collection

### Current State: Freezer Does NOT Have Native Ceilometer Integration

**IMPORTANT**: As of the latest OpenStack release (2025.1), **Freezer does not have built-in Ceilometer integration** or predefined meters in the official Ceilometer measurements catalog. 

The official Ceilometer documentation lists meters for Nova, Cinder, Glance, Swift, Neutron, and other core services, but **Freezer is not included**.

### What This Means for Billing

To implement billing for Freezer, you have **three options**:

#### Option 1: Custom Meter Definitions (Recommended)

You must create custom Ceilometer meter definitions that consume Freezer notifications. This requires:

1. **Freezer must emit notifications** to RabbitMQ (this needs to be verified/implemented in Freezer code)
2. **Custom meter definitions** in `/etc/ceilometer/meters.d/` to parse Freezer notifications
3. **Custom event definitions** in `/etc/ceilometer/event_definitions.yaml`
4. **Custom Gnocchi resource types** for backup resources

#### Option 2: Direct API Polling

Create custom Ceilometer pollsters that query the Freezer API directly:

- Poll Freezer API for backup job status
- Calculate storage consumption from backup metadata
- Track restore operations via API logs

#### Option 3: External Metering System

Use an external system (Prometheus, custom scripts) to:

- Query Freezer API periodically
- Export metrics to CloudKitty directly
- Bypass Ceilometer entirely

---

<!-- ### Proposed Custom Metrics for Freezer (Requires Implementation)

Below are **proposed** metrics that would need to be implemented via custom Ceilometer configuration. These are NOT currently available out-of-the-box:

#### 1. **Storage Metrics** (Primary Billing Metrics - PROPOSED)

| Metric Name | Type | Unit | Description | Implementation Status |
|-------------|------|------|-------------|----------------------|
| `backup.size` | Gauge | GB | Total size of backup data stored | **Requires custom meter definition** |
| `backup.compressed_size` | Gauge | GB | Actual compressed backup size | **Requires custom meter definition** |
| `backup.dedup_ratio` | Gauge | Ratio | Deduplication efficiency ratio | **Requires custom meter definition** |
| `backup.storage.duration` | Gauge | GB-hours | Storage size × time stored | **Requires custom meter definition** |
| `backup.storage.tier` | String | - | Storage tier (hot/warm/cold/archive) | **Requires custom meter definition** |

**Note**: These metrics require Freezer to emit notifications with the appropriate payload data.

#### 2. **Operation Metrics** (Activity Tracking - PROPOSED)

| Metric Name | Type | Unit | Description | Implementation Status |
|-------------|------|------|-------------|----------------------|
| `backup.create.count` | Delta | Count | Number of backup operations | **Requires custom meter definition** |
| `backup.restore.count` | Delta | Count | Number of restore operations | **Requires custom meter definition** |
| `backup.delete.count` | Delta | Count | Number of backup deletions | **Requires custom meter definition** |
| `backup.verify.count` | Delta | Count | Number of backup verifications | **Requires custom meter definition** |
| `backup.operation.duration` | Gauge | Seconds | Time taken for backup/restore | **Requires custom meter definition** |

#### 3. **Data Transfer Metrics** (Bandwidth Tracking)

| Metric Name | Type | Unit | Description | Billing Impact |
|-------------|------|------|-------------|----------------|
| `backup.upload.bytes` | Delta | Bytes | Data uploaded during backup | Medium - Ingress bandwidth |
| `backup.download.bytes` | Delta | Bytes | Data downloaded during restore | **HIGH** - Egress bandwidth charges |
| `backup.network.bandwidth` | Gauge | Mbps | Network bandwidth utilized | Medium - QoS and capacity planning |

#### 4. **Resource-Specific Metrics**

| Metric Name | Type | Unit | Description | Billing Impact |
|-------------|------|------|-------------|----------------|
| `backup.vm.count` | Gauge | Count | Number of VMs backed up | Medium - Per-VM charges |
| `backup.volume.count` | Gauge | Count | Number of volumes backed up | Medium - Per-volume charges |
| `backup.database.count` | Gauge | Count | Number of databases backed up | Medium - Per-database charges |
| `backup.filesystem.count` | Gauge | Count | Number of filesystems backed up | Low - Typically bundled |

<!-- #### 5. **Performance and Quality Metrics**

| Metric Name | Type | Unit | Description | Billing Impact |
|-------------|------|------|-------------|----------------|
| `backup.success.rate` | Gauge | Percentage | Backup success rate | None - SLA monitoring |
| `backup.failure.count` | Delta | Count | Number of failed backups | None - Alerting only |
| `backup.compression.ratio` | Gauge | Ratio | Compression efficiency | None - Optimization metric |
| `backup.throughput` | Gauge | MB/s | Backup/restore throughput | None - Performance tracking | -->

<!-- #### 5. **API and HTTP Metrics**

| Metric Name | Type | Unit | Description | Billing Impact |
|-------------|------|------|-------------|----------------|
| `backup.api.requests` | Delta | Count | Total API requests | Low - Per-request charges (optional) |
| `backup.api.get` | Delta | Count | HTTP GET requests | Low - Read operations |
| `backup.api.post` | Delta | Count | HTTP POST requests | Low - Create operations |
| `backup.api.put` | Delta | Count | HTTP PUT requests | Low - Update operations |
| `backup.api.delete` | Delta | Count | HTTP DELETE requests | Low - Delete operations |
| `backup.api.response_time` | Gauge | Milliseconds | API response time | None - Performance monitoring |
| `backup.api.errors` | Delta | Count | API error count | None - Health monitoring | -->



## Implementation Guide: Adding Freezer Metrics to Ceilometer

Since Freezer doesn't have native Ceilometer integration, here's how to implement it:

### Step 1: Verify Freezer Notification Capability

First, check if Freezer emits notifications to RabbitMQ. You may need to:

1. Check Freezer configuration for notification settings
2. Enable notification drivers in `/etc/freezer/freezer-api.conf`:
   ```ini
   [oslo_messaging_notifications]
   driver = messagingv2
   topics = notifications
   ```

3. Verify notifications are being sent:
   ```bash
   rabbitmqctl list_queues -p openstack | grep freezer
   ```

### Step 2: Create Custom Meter Definitions

Create `/etc/ceilometer/meters.d/freezer.yaml` with proposed meter definitions:

```yaml
---
# PROPOSED Freezer meter definitions
# These require Freezer to emit appropriate notifications

- name: 'backup.size'
  event_type:
    - 'freezer.backup.create.end'
    - 'freezer.backup.update'
  type: 'gauge'
  unit: 'GB'
  volume: $.payload.backup_size_gb
  user_id: $.payload.user_id
  project_id: $.payload.project_id
  resource_id: $.payload.backup_id
  metadata:
    backup_name: $.payload.backup_name
    backup_type: $.payload.backup_type
    source_type: $.payload.source_type

- name: 'backup.restore.size'
  event_type:
    - 'freezer.restore.end'
  type: 'delta'
  unit: 'GB'
  volume: $.payload.restored_size_gb
  user_id: $.payload.user_id
  project_id: $.payload.project_id
  resource_id: $.payload.backup_id
```

**IMPORTANT**: The event types (`freezer.backup.create.end`, etc.) and payload fields (`$.payload.backup_size_gb`, etc.) are **hypothetical**. We must verify what Freezer actually emits by inspecting RabbitMQ messages.

### Step 3: Alternative - Direct API Polling

If Freezer doesn't emit notifications, create a custom pollster:

```python
# /usr/lib/python3/dist-packages/ceilometer/backup/pollsters.py
from ceilometer import pollster
from ceilometer import sample
import requests

class BackupSizePollster(pollster.BasePolls):
    def get_samples(self, manager, cache, resources):
        # Query Freezer API
        freezer_api = "http://freezer-api:9090/v1/backups"
        response = requests.get(freezer_api)
        backups = response.json()
        
        for backup in backups:
            yield sample.Sample(
                name='backup.size',
                type=sample.TYPE_GAUGE,
                unit='GB',
                volume=backup.get('size_gb', 0),
                user_id=backup.get('user_id'),
                project_id=backup.get('project_id'),
                resource_id=backup.get('id'),
                resource_metadata=backup
            )
```

### Step 4: Alternative - Use Existing Cinder/Swift Metrics

Since Freezer backs up to Swift or Cinder, we can leverage **existing** Ceilometer metrics:

**For Swift-backed Freezer backups**, use these REAL metrics:
- `storage.objects.size` - Total size of objects in Swift (includes Freezer backups)
- `storage.objects` - Number of objects
- `storage.objects.incoming.bytes` - Upload bandwidth
- `storage.objects.outgoing.bytes` - Download bandwidth (restore operations)

**For Cinder-backed Freezer backups**, use these REAL metrics:
- `volume.size` - Size of Cinder volumes used for backups
- `snapshot.size` - Size of Cinder snapshots

**Implementation**:
```bash
# Query Swift metrics for Freezer backup container
openstack metric measures show \
  --resource-id <swift_account_id> \
  --aggregation mean \
  storage.objects.size

# Filter by Freezer backup container in metadata
openstack metric resource show <resource_id> \
  | grep "freezer-backup"
```

This approach uses **real, existing Ceilometer metrics** but requires:
- Freezer backups stored in dedicated Swift containers or Cinder volumes
- Tagging/labeling to distinguish Freezer data from other Swift/Cinder usage

---

<!-- ## Detailed Metric Collection Examples (PROPOSED)

**DISCLAIMER**: The following examples show how metrics COULD be collected IF Freezer had native Ceilometer integration. These are templates for implementation, not existing functionality.

### Example 1: Tracking Backup Storage (PROPOSED)

```yaml
# PROPOSED Ceilometer meter definition
- name: 'backup.storage.duration'
  event_type:
    - 'freezer.backup.exists'  # HYPOTHETICAL - verify actual event type
  type: 'gauge'
  unit: 'GB-hours'
  volume: $.payload.backup_size_gb  # HYPOTHETICAL - verify actual payload field
  user_id: $.payload.user_id
  project_id: $.payload.project_id
  resource_id: $.payload.backup_id
  metadata:
    backup_name: $.payload.backup_name
    storage_tier: $.payload.storage_tier
    age_days: $.payload.age_days
    compression_ratio: $.payload.compression_ratio
    dedup_ratio: $.payload.dedup_ratio
```

**Status**: PROPOSED - Requires Freezer to emit notifications with this payload structure

**What This Would Track**: 
- Backup size in GB multiplied by hours stored
- Would enable hourly billing: `Cost = GB-hours × $0.000017/GB-hour`

### Example 2: Tracking Restore Operations (PROPOSED)

```yaml
# PROPOSED Ceilometer meter definition
- name: 'backup.restore.size'
  event_type:
    - 'freezer.restore.end'  # HYPOTHETICAL - verify actual event type
  type: 'delta'
  unit: 'GB'
  volume: $.payload.restored_size_gb  # HYPOTHETICAL - verify actual payload field
  user_id: $.payload.user_id
  project_id: $.payload.project_id
  resource_id: $.payload.backup_id
  metadata:
    restore_type: $.payload.restore_type
    restore_destination: $.payload.destination
    restore_duration: $.payload.duration_seconds
    success: $.payload.success
```

**Status**: PROPOSED - Requires Freezer to emit notifications with this payload structure

**What This Would Track**:
- Amount of data restored in GB
- Would enable per-restore billing: `Cost = GB restored × $0.004/GB` -->

### Example: Using REAL Swift Metrics for Freezer Backups

```yaml
# EXISTING Ceilometer meter (already available)
- name: 'storage.objects.size'
  # This is a REAL metric that already exists in Ceilometer
  # Can be used to track Freezer backup storage if backups go to Swift
```

**Query Example**:
```bash
# Get storage size for Freezer backup container
openstack metric measures show \
  --resource-id <swift_account>:<freezer_container> \
  --aggregation sum \
  storage.objects.size \
  --start 2026-01-01

# Get egress bandwidth (restore operations)
openstack metric measures show \
  --resource-id <swift_account>:<freezer_container> \
  --aggregation sum \
  storage.objects.outgoing.bytes \
  --start 2026-01-01
```

**Status**: AVAILABLE NOW - Uses existing Swift metrics

**What This Tracks**:
- Actual storage consumption in Swift (where Freezer stores backups)
- Egress bandwidth during restores
- Can be used for billing TODAY without custom development

---

## Practical Billing Implementation (Using Existing Metrics)

### Recommended Approach: Leverage Swift/Cinder Metrics

Since Freezer stores backups in Swift or Cinder, use the **existing, real** Ceilometer metrics:

#### For Swift-Backed Freezer Deployments:

1. **Storage Billing**:
   - Metric: `storage.objects.size` (REAL, available now)
   - Filter by Freezer container name
   - Bill per GB-month

2. **Restore Billing**:
   - Metric: `storage.objects.outgoing.bytes` (REAL, available now)
   - Represents data downloaded during restore
   - Bill per GB transferred

3. **Backup Operations**:
   - Metric: `storage.objects.incoming.bytes` (REAL, available now)
   - Represents data uploaded during backup
   - Optional billing per GB uploaded

#### For Cinder-Backed Freezer Deployments:

1. **Storage Billing**:
   - Metric: `volume.size` (REAL, available now)
   - Filter by Freezer-tagged volumes
   - Bill per GB-month

2. **Snapshot Billing**:
   - Metric: `snapshot.size` (REAL, available now)
   - Incremental backup snapshots
   - Bill per GB-month

### CloudKitty Configuration (Using Real Metrics)

Need to check AtomHopper/UMS compliant configuration here.

```yaml
# CloudKitty rating for Freezer using REAL Swift metrics
services:
  freezer_backup_storage:
    metrics:
      - metric: storage.objects.size
        groupby:
          - container  # Filter for freezer-backup-* containers
        rating_type: flat
        cost_per_unit: 0.012  # $/GB/month
        unit: GB
        
  freezer_restore_bandwidth:
    metrics:
      - metric: storage.objects.outgoing.bytes
        groupby:
          - container  # Filter for freezer-backup-* containers
        rating_type: flat
        cost_per_unit: 0.004  # $/GB
        unit: GB
```

This configuration uses **real, existing metrics** and works TODAY without custom development.

---

## Summary: What Metrics Are Actually Available?

### Available NOW (Real Ceilometer Metrics)

If Freezer uses Swift for backup storage:
- `storage.objects.size` - Total backup storage
- `storage.objects` - Number of backup objects
- `storage.objects.incoming.bytes` - Backup upload bandwidth
- `storage.objects.outgoing.bytes` - Restore download bandwidth
- `storage.containers.objects.size` - Per-container storage

If Freezer uses Cinder for backup storage:
- `volume.size` - Backup volume size
- `snapshot.size` - Backup snapshot size

### NOT Available (Requires Custom Implementation)

- `backup.size` - Freezer-specific backup size tracking
- `backup.restore.count` - Number of restore operations
- `backup.operation.duration` - Backup/restore timing
- `backup.api.requests` - Freezer API usage
- Any Freezer-specific metrics

### Requires Development

To get Freezer-specific metrics, you need to:
1. Add notification emitters to Freezer code
2. Create custom Ceilometer meter definitions
3. OR create custom pollsters that query Freezer API
4. OR use external monitoring (Prometheus + custom exporters)

---

<!-- ## Public Cloud Equivalents

| OpenStack Component | AWS Equivalent | Azure Equivalent | GCP Equivalent | Primary Function |
|---------------------|----------------|------------------|----------------|------------------|
| **Freezer** | AWS Backup | Azure Backup | Cloud Backup and DR | Backup and disaster recovery service |

---

## Freezer Backup Pricing Model

### Public Cloud Backup Pricing (Reference Rates)

#### AWS Backup Pricing (US East - January 2026)

| Service | Storage Cost | Restore Cost | Additional Fees |
|---------|--------------|--------------|-----------------|
| **Warm Storage** | $0.05/GB/month | $0.02/GB restored | - |
| **Cold Storage** | $0.01/GB/month | $0.05/GB restored | - |
| **EBS Snapshots** | $0.05/GB/month | Included | - |
| **RDS Snapshots** | $0.095/GB/month | Included | - |
| **VM Backup** | $0.05/GB/month | $0.02/GB restored | - |

#### Azure Backup Pricing (US East - January 2026)

| Service | Storage Cost | Restore Cost | Additional Fees |
|---------|--------------|--------------|-----------------|
| **VM Backup** | $0.10/instance/month + $0.05/GB | Free | - |
| **Database Backup** | $0.10/GB/month | Free | - |
| **File/Folder** | $0.05/GB/month | Free | Bandwidth charges apply |

#### GCP Backup Pricing (US Central - January 2026)

| Service | Storage Cost | Restore Cost | Additional Fees |
|---------|--------------|--------------|-----------------|
| **VM Snapshots** | $0.026/GB/month | Included | - |
| **Persistent Disk Snapshots** | $0.026/GB/month | Included | - |
| **Database Backups** | $0.08/GB/month | Included | - |

---

## OpenStack Freezer Flex Pricing Strategy

### Discount Justification for OpenStack-Based Pricing

When running OpenStack Freezer on your own infrastructure, the following discounts are justified:

1. **Infrastructure Ownership Discount (30-40%)**: We own and operate the hardware, eliminating cloud provider markup
2. **No Egress Fees Discount (10-15%)**: No data transfer charges for restores within your datacenter
3. **Multi-tenancy Efficiency (15-20%)**: Shared infrastructure across multiple projects/departments
4. **Long-term Commitment (10-15%)**: Capital investment in infrastructure vs. operational expenses
5. **No API Call Charges (5-10%)**: Unlimited backup/restore operations without per-request fees

**Total Justified Discount Range: 70-80% compared to public cloud pricing**

### Recommended OpenStack Freezer Pricing

| Backup Type | Storage Cost (Monthly) | Storage Cost (Hourly) | Storage Cost (Yearly) | Restore Cost | Notes |
|-------------|------------------------|------------------------|------------------------|--------------|-------|
| **MongoDB Backup** | $0.012/GB | $0.000017/GB | $0.144/GB | $0.004/GB | Compressed, incremental |
| **MySQL Backup** | $0.012/GB | $0.000017/GB | $0.144/GB | $0.004/GB | Compressed, incremental |
| **Cinder Volume Backup** | $0.010/GB | $0.000014/GB | $0.120/GB | $0.003/GB | Block-level incremental |
| **OpenStack VM Backup** | $0.015/GB | $0.000021/GB | $0.180/GB | $0.005/GB | Full VM snapshot |
| **Local Folder Backup** | $0.008/GB | $0.000011/GB | $0.096/GB | $0.002/GB | File-level backup |

### Tiered Storage Pricing (Optional)

| Storage Tier | Cost Multiplier | Restore Time | Use Case |
|--------------|-----------------|--------------|----------|
| **Hot Storage** | 1.0x base price | Immediate | Recent backups (0-30 days) |
| **Warm Storage** | 0.6x base price | < 1 hour | Medium-term (31-90 days) |
| **Cold Storage** | 0.3x base price | < 4 hours | Long-term (90+ days) |
| **Archive Storage** | 0.15x base price | < 24 hours | Compliance (1+ years) |

---  -->

## Ceilometer Metrics for Freezer Billing

### Recommended Meter Definitions

The following Ceilometer meters should be configured to track Freezer usage for billing purposes:

```yaml
# Freezer Backup Storage Meters
- name: 'backup.size'
  event_type:
    - 'freezer.backup.create.end'
    - 'freezer.backup.update'
  type: 'gauge'
  unit: 'GB'
  volume: $.payload.backup_size_gb
  user_id: $.payload.user_id
  project_id: $.payload.project_id
  resource_id: $.payload.backup_id
  metadata:
    backup_name: $.payload.backup_name
    backup_type: $.payload.backup_type
    source_type: $.payload.source_type
    storage_tier: $.payload.storage_tier
    compression_ratio: $.payload.compression_ratio

- name: 'backup.restore.size'
  event_type:
    - 'freezer.restore.end'
  type: 'delta'
  unit: 'GB'
  volume: $.payload.restored_size_gb
  user_id: $.payload.user_id
  project_id: $.payload.project_id
  resource_id: $.payload.backup_id
  metadata:
    restore_destination: $.payload.destination
    restore_duration: $.payload.duration_seconds

- name: 'backup.storage.duration'
  event_type:
    - 'freezer.backup.exists'
  type: 'gauge'
  unit: 'GB-hours'
  volume: $.payload.backup_size_gb
  user_id: $.payload.user_id
  project_id: $.payload.project_id
  resource_id: $.payload.backup_id
  metadata:
    age_days: $.payload.age_days
    storage_tier: $.payload.storage_tier

- name: 'backup.operation.count'
  event_type:
    - 'freezer.backup.create.end'
    - 'freezer.restore.end'
    - 'freezer.backup.delete.end'
  type: 'delta'
  unit: 'operation'
  volume: 1
  user_id: $.payload.user_id
  project_id: $.payload.project_id
  resource_id: $.payload.backup_id
  metadata:
    operation_type: $.payload.operation_type
    success: $.payload.success
```

### Gnocchi Resource Definition for Freezer

```yaml
- resource_type: backup
  metrics:
    backup.size:
    backup.restore.size:
    backup.storage.duration:
    backup.operation.count:
  attributes:
    backup_name: resource_metadata.backup_name
    backup_type: resource_metadata.backup_type
    source_type: resource_metadata.source_type
    storage_tier: resource_metadata.storage_tier
    compression_ratio: resource_metadata.compression_ratio
  event_create:
    - freezer.backup.create.end
  event_delete:
    - freezer.backup.delete.end
  event_update:
    - freezer.backup.update
  event_attributes:
    id: resource_id
    project_id: project_id
    user_id: user_id
```

---

<!-- ## Detailed Use Cases with Pricing Examples

### Use Case 1: MongoDB Database Backup

**Scenario**: Daily backup of a 500GB MongoDB database with 7-day retention

**Configuration**:
- Database size: 500 GB
- Backup frequency: Daily (full weekly, incremental daily)
- Retention: 7 days
- Compression ratio: 3:1
- Average compressed backup size: 167 GB (full), 20 GB (incremental)

**Monthly Storage Calculation**:
```
Week 1: 1 full (167 GB) + 6 incremental (120 GB) = 287 GB
Week 2-4: Same pattern = 287 GB × 3 = 861 GB
Average storage: (287 + 861) / 2 = 574 GB (accounting for rotation)

Monthly cost: 574 GB × $0.012/GB = $6.89/month
Yearly cost: $6.89 × 12 = $82.68/year
```

**Restore Scenario** (once per quarter):
```
Restore cost: 167 GB × $0.004/GB = $0.67 per restore
Quarterly restore cost: $0.67 × 1 = $0.67
Annual restore cost: $0.67 × 4 = $2.68
```

**Total Annual Cost**: $82.68 + $2.68 = **$85.36**

**Comparison to AWS Backup**: 
- AWS cost: 574 GB × $0.095/GB × 12 = $654.36/year + restore fees
- **Savings: $569.00/year (87% reduction)**

### Use Case 2: MySQL Database Backup

**Scenario**: Hourly incremental backups of a 200GB MySQL database with 30-day retention

**Configuration**:
- Database size: 200 GB
- Backup frequency: Hourly incremental, daily full
- Retention: 30 days
- Compression ratio: 4:1
- Average compressed backup: 50 GB (full), 2 GB (incremental)

**Monthly Storage Calculation**:
```
Daily: 1 full (50 GB) + 23 incremental (46 GB) = 96 GB
30-day retention: 96 GB × 30 = 2,880 GB
With deduplication (70% efficiency): 2,880 GB × 0.3 = 864 GB

Monthly cost: 864 GB × $0.012/GB = $10.37/month
Hourly cost: $10.37 / 730 hours = $0.014/hour
Yearly cost: $10.37 × 12 = $124.44/year
```

**Restore Scenario** (monthly testing):
```
Restore cost: 50 GB × $0.004/GB = $0.20 per restore
Monthly restore cost: $0.20 × 1 = $0.20
Annual restore cost: $0.20 × 12 = $2.40
```

**Total Annual Cost**: $124.44 + $2.40 = **$126.84**

**Comparison to Azure Backup**:
- Azure cost: 864 GB × $0.10/GB × 12 = $1,036.80/year
- **Savings: $909.96/year (88% reduction)**

### Use Case 3: Cinder Volume Backup

**Scenario**: Weekly backup of 10 Cinder volumes (100GB each) with 90-day retention

**Configuration**:
- Total volume size: 1,000 GB (10 × 100 GB)
- Backup frequency: Weekly full, daily incremental
- Retention: 90 days (13 weeks)
- Block-level incremental: ~10% change rate
- Average backup: 100 GB (full), 10 GB (incremental)

**Monthly Storage Calculation**:
```
Per volume weekly: 1 full (100 GB) + 6 incremental (60 GB) = 160 GB
All volumes weekly: 160 GB × 10 = 1,600 GB
13-week retention: 1,600 GB × 13 = 20,800 GB
With block-level dedup (80% efficiency): 20,800 GB × 0.2 = 4,160 GB

Monthly cost: 4,160 GB × $0.010/GB = $41.60/month
Yearly cost: $41.60 × 12 = $499.20/year
```

**Restore Scenario** (quarterly DR test):
```
Restore cost: 1,000 GB × $0.003/GB = $3.00 per restore
Quarterly restore cost: $3.00 × 1 = $3.00
Annual restore cost: $3.00 × 4 = $12.00
```

**Total Annual Cost**: $499.20 + $12.00 = **$511.20**

**Comparison to AWS EBS Snapshots**:
- AWS cost: 4,160 GB × $0.05/GB × 12 = $2,496.00/year
- **Savings: $1,984.80/year (79% reduction)**

### Use Case 4: OpenStack VM Backup

**Scenario**: Daily backup of 50 VMs (average 80GB each) with 14-day retention

**Configuration**:
- Total VM storage: 4,000 GB (50 × 80 GB)
- Backup frequency: Daily full snapshot
- Retention: 14 days
- Compression ratio: 2:1
- Average compressed backup: 40 GB per VM

**Monthly Storage Calculation**:
```
Daily backup: 50 VMs × 40 GB = 2,000 GB
14-day retention: 2,000 GB × 14 = 28,000 GB

Monthly cost: 28,000 GB × $0.015/GB = $420.00/month
Yearly cost: $420.00 × 12 = $5,040.00/year
```

**Restore Scenario** (weekly for testing/recovery):
```
Average restore: 5 VMs × 40 GB = 200 GB per week
Restore cost: 200 GB × $0.005/GB = $1.00 per week
Monthly restore cost: $1.00 × 4 = $4.00
Annual restore cost: $4.00 × 12 = $48.00
```

**Total Annual Cost**: $5,040.00 + $48.00 = **$5,088.00**

**Comparison to AWS Backup**:
- AWS cost: 28,000 GB × $0.05/GB × 12 = $16,800.00/year + restore fees ($0.02/GB)
- AWS restore: 200 GB × 52 weeks × $0.02/GB = $208.00/year
- AWS total: $17,008.00/year
- **Savings: $11,920.00/year (70% reduction)**

### Use Case 5: Local Folder Backup (User Data)

**Scenario**: Continuous backup of 1,000 user home directories (average 50GB each) with 60-day retention

**Configuration**:
- Total data: 50,000 GB (1,000 × 50 GB)
- Backup frequency: Continuous (file-level changes)
- Retention: 60 days
- Deduplication ratio: 5:1 (high redundancy in user files)
- Average unique data: 10,000 GB

**Monthly Storage Calculation**:
```
Unique data with dedup: 10,000 GB
60-day retention with versioning: 10,000 GB × 1.5 = 15,000 GB

Monthly cost: 15,000 GB × $0.008/GB = $120.00/month
Yearly cost: $120.00 × 12 = $1,440.00/year
```

**Restore Scenario** (daily user requests):
```
Average daily restore: 10 users × 5 GB = 50 GB per day
Monthly restore: 50 GB × 30 = 1,500 GB
Restore cost: 1,500 GB × $0.002/GB = $3.00/month
Annual restore cost: $3.00 × 12 = $36.00
```

**Total Annual Cost**: $1,440.00 + $36.00 = **$1,476.00**

**Comparison to Azure Backup**:
- Azure cost: 15,000 GB × $0.05/GB × 12 = $9,000.00/year
- **Savings: $7,524.00/year (84% reduction)**

---

## Cost Summary Table

| Use Case | Data Size | Monthly Cost | Yearly Cost | Public Cloud Equivalent | Annual Savings | Savings % |
|----------|-----------|--------------|-------------|-------------------------|----------------|-----------|
| MongoDB Backup (500GB) | 574 GB avg | $6.89 | $85.36 | $654.36 (AWS) | $569.00 | 87% |
| MySQL Backup (200GB) | 864 GB avg | $10.37 | $126.84 | $1,036.80 (Azure) | $909.96 | 88% |
| Cinder Volumes (1TB) | 4,160 GB avg | $41.60 | $511.20 | $2,496.00 (AWS) | $1,984.80 | 79% |
| VM Backup (4TB) | 28,000 GB | $420.00 | $5,088.00 | $17,008.00 (AWS) | $11,920.00 | 70% |
| User Folders (50TB) | 15,000 GB | $120.00 | $1,476.00 | $9,000.00 (Azure) | $7,524.00 | 84% |
| **TOTAL** | - | **$599.86** | **$7,287.40** | **$30,195.16** | **$22,907.76** | **76%** |

--- -->

## Implementation Recommendations

### 1. Tiered Storage Strategy

Implement automatic lifecycle policies to move backups between storage tiers:

```yaml
# Example Freezer storage policy
storage_policies:
  - name: "hot-to-warm"
    age_days: 30
    source_tier: "hot"
    destination_tier: "warm"
    
  - name: "warm-to-cold"
    age_days: 90
    source_tier: "warm"
    destination_tier: "cold"
    
  - name: "cold-to-archive"
    age_days: 365
    source_tier: "cold"
    destination_tier: "archive"
```

### 2. Compression and Deduplication

Enable aggressive compression and deduplication to maximize storage efficiency:

- **MongoDB/MySQL**: Use native database compression tools before Freezer backup
- **Cinder Volumes**: Enable block-level deduplication
- **VMs**: Use thin provisioning and exclude swap/temp files
- **User Folders**: Enable file-level deduplication across users

### 3. Backup Scheduling Optimization

Optimize backup windows to reduce resource contention:

```yaml
# Example backup schedule
backup_schedules:
  databases:
    full_backup: "0 2 * * 0"  # Sunday 2 AM
    incremental: "0 */6 * * *"  # Every 6 hours
    
  volumes:
    full_backup: "0 3 * * 0"  # Sunday 3 AM
    incremental: "0 1 * * *"  # Daily 1 AM
    
  vms:
    snapshot: "0 4 * * *"  # Daily 4 AM
    
  user_folders:
    continuous: true
    throttle_mbps: 100
```

### 4. Monitoring and Alerting

Configure Ceilometer alarms for backup health and cost management:

```yaml
# Example Ceilometer alarm for backup costs
alarms:
  - name: "backup-cost-threshold"
    meter: "backup.storage.duration"
    threshold: 50000  # GB-hours
    comparison: "gt"
    period: 2592000  # 30 days
    evaluation_periods: 1
    statistic: "sum"
    
  - name: "backup-failure-rate"
    meter: "backup.operation.count"
    threshold: 0.95  # 95% success rate
    comparison: "lt"
    period: 86400  # 24 hours
    evaluation_periods: 1
    statistic: "avg"
```

<!-- ---

## Billing Integration with CloudKitty

To implement automated billing based on Freezer usage, integrate with CloudKitty:

```yaml
# CloudKitty rating configuration for Freezer
services:
  backup:
    metrics:
      - metric: backup.size
        rating_type: flat
        cost_per_unit: 0.012  # $/GB/month
        unit: GB
        
      - metric: backup.restore.size
        rating_type: flat
        cost_per_unit: 0.004  # $/GB restored
        unit: GB
        
      - metric: backup.storage.duration
        rating_type: tiered
        tiers:
          - threshold: 0
            cost_per_unit: 0.000017  # $/GB/hour (hot)
          - threshold: 720  # 30 days
            cost_per_unit: 0.00001  # $/GB/hour (warm)
          - threshold: 2160  # 90 days
            cost_per_unit: 0.000005  # $/GB/hour (cold)
``` 

--- -->

## Conclusion

OpenStack Freezer provides significant cost savings compared to public cloud backup services while maintaining enterprise-grade backup and recovery capabilities. The combination of infrastructure ownership, no egress fees, and efficient multi-tenancy justifies the substantial discount over public cloud pricing.

By implementing proper Ceilometer metering and billing integration, organizations can accurately track and charge back backup costs to individual projects or departments, creating a sustainable internal cloud economics model.

### Key Takeaways

1. **Significant savings** compared to public cloud backup services
2. **Predictable pricing** with no surprise egress or API call charges
3. **Flexible tiered storage** for cost optimization
<!-- 4. **Comprehensive metering** via Ceilometer for accurate billing
5. **Automated lifecycle management** to minimize storage costs -->

### Next Steps

1. Deploy Freezer using the installation guide
2. Configure Ceilometer metrics for backup tracking
3. Integrate Atom Hopper+UMS solution for automated billing
4. Set up tiered storage policies
5. Monitor and optimize backup efficiency
6. Review and adjust pricing quarterly based on actual costs

---

## References

- [OpenStack Freezer Documentation](https://docs.openstack.org/freezer/latest/)
- [Ceilometer Metering Guide](https://docs.openstack.org/ceilometer/latest/)
- [CloudKitty Rating Documentation](https://docs.openstack.org/cloudkitty/latest/)
- [AWS Backup Pricing](https://aws.amazon.com/backup/pricing/)
- [Azure Backup Pricing](https://azure.microsoft.com/en-us/pricing/details/backup/)
- [GCP Backup Pricing](https://cloud.google.com/compute/disks-image-pricing)
