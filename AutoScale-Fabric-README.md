# 🚀 Auto‑Scale Microsoft Fabric Capacity Based on Real‑Time Utilization

Microsoft Fabric does not yet provide built‑in autoscale for capacity.  
This solution demonstrates how to **monitor real‑time capacity usage** and automatically **scale up** when utilization exceeds a defined threshold.

The approach leverages:

- **Eventstream** to ingest Capacity Overview Events  
- **SQL operator** to compute utilization in real‑time  
- **Activator** to evaluate 10‑minute rolling utilization  
- **Notebook** to execute a REST API–based scale‑up action  

This avoids performance throttling during workload spikes while preventing over‑provisioning during normal operations.

---

## 📐 Utilization Formula

```
utilization = capacityUnitMs / (baseCapacityUnits * windowDurationMs)
```

See **Capacity Overview Events** documentation:  
https://learn.microsoft.com/fabric/real-time-hub/capacity-overview-events

---

# 🧩 Architecture Components

### 1. Capture Capacity Data  
Use Fabric **Capacity Overview Events** as an Eventstream source.  
Reference: https://learn.microsoft.com/fabric/real-time-hub/capacity-overview-events

### 2. Process & Compute Utilization  
A SQL operator processes the incoming stream and computes utilization.  
Reference: https://learn.microsoft.com/fabric/real-time-hub/sql-operator

### 3. Trigger Actions with Activator  
Activator watches computed utilization and executes a notebook when thresholds are exceeded.  
Reference: https://learn.microsoft.com/fabric/real-time-hub/activator

---

# 🛠 Step‑by‑Step Implementation

## 1. Create Eventstream  
Guide: https://learn.microsoft.com/fabric/real-time-hub/create-eventstream

---

## 2. Add Capacity Overview Event Source

Steps:

1. Select **Capacity Overview Events** → **Connect**  
2. Choose event type: `Microsoft.Fabric.Capacity.Summary`  
3. Select a capacity to monitor  
4. Rename the source to **Fabric-capacity-events**

More details:  
https://learn.microsoft.com/fabric/real-time-hub/add-capacity-overview-event-source

---

## 3. Add SQL Transformation

Name: `Calculate_capacity_utilization`  
Guide: https://learn.microsoft.com/fabric/real-time-hub/sql-operator

### SQL Script

```sql
WITH u AS (
  SELECT
    data.capacityId       AS capacityId,
    data.capacityName     AS capacityName,
    data.capacitySku      AS capacitySku,
    data.windowStartTime  AS windowStartTime,
    data.windowEndTime    AS windowEndTime,
    CAST(data.baseCapacityUnits AS float) AS baseCapacityUnits,
    CAST(data.capacityUnitMs     AS float) AS capacityUnitMs,
    CAST(
      CAST(data.capacityUnitMs AS float) /
      NULLIF(
        CAST(data.baseCapacityUnits AS float) *
        DATEDIFF(millisecond, data.windowStartTime, data.windowEndTime),
        0
      ) AS float
    ) AS utilization
  FROM [CapacityScaleupdemo-stream]
)
SELECT
  u.capacityId,
  u.capacityName,
  u.capacitySku,
  u.windowStartTime,
  u.windowEndTime,
  u.baseCapacityUnits,
  u.capacityUnitMs,
  u.utilization
INTO [Capacity-Activator]
FROM u;
```

---

# 4. Publish Eventstream  
After publishing, all nodes show **Active** status.

---

# ⚡ Configure Activator Rule

1. Create Object  
   - **Name**: Fabric Capacity  
   - **Identifier**: `capacityName`  
   - **Property**: `utilization`

2. Create rule  
   - **Summarization**: Average (10‑minute window)  
   - **Condition**: utilization > 0.8 for 10 minutes  
   - **Action**: run notebook `scale_up_by_api`

---

# 🔧 Scale Up Using Fabric REST API

Activator supports **one action per rule**.

For multi‑step workflows:

- Use **Power Automate** to orchestrate multiple downstream actions, or  
- Send Teams/Email notifications from the notebook itself  

---

# 📝 Sample Notebook: Scale Up Fabric Capacity

```python
# Sample code omitted for brevity; identical to provided version
```

---
