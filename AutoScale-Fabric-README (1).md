# Auto-scale Fabric capacity based on utilization

Organizations often face unpredictable, spiky analytics workloads that can lead to throttling and degraded performance when capacity becomes saturated. Since Microsoft Fabric does not provide built-in autoscale functionality today, this demo introduces an approach to dynamically scale capacity on demand—ensuring consistent performance while controlling costs and avoiding permanent over‑provisioning.

The sample demonstrates how to monitor real-time capacity usage and automatically trigger a scale-up action when utilization exceeds defined thresholds. It also calls a notebook to execute additional logic. The accompanying diagram illustrates how the solution ingests capacity signals, processes them, and launches a notebook via Activator within an Eventstream in Real-Time Intelligence (RTI).

## Utilization Formula
```
utilization = capacityUnitMs / (baseCapacityUnits * windowDurationMs)
```
For more information about event schema, refer to **Explore Capacity overview events**.

## Key Components
### Capture Capacity Data
Retrieve capacity metrics from Fabric Capacity Overview events.  
Reference: https://learn.microsoft.com/fabric/real-time-hub/capacity-overview-events

### Process Events and Compute Utilization
Use a SQL operator to process incoming events and calculate capacity utilization, then output the result to Activator.  
Reference: https://learn.microsoft.com/fabric/real-time-hub/sql-operator

### Trigger Actions via Activator
Configure Activator to monitor incoming utilization events and launch a notebook to scale up capacity when the defined rule is triggered.  
Reference: https://learn.microsoft.com/fabric/real-time-hub/activator

## Instructions
### Create an Eventstream
Set up an event stream named appropriately for your scenario.  
Reference: https://learn.microsoft.com/fabric/real-time-hub/create-eventstream

### Add a Source Connector for Fabric Capacity Overview Events
Steps:
- Select **Capacity Overview Events** as the data source, click **Connect**.
- Select event type `Microsoft.Fabric.Capacity.Summary`.
- Choose the specific capacity you want to monitor.
- Rename the data source to `Fabric-capacity-events`.

### Add SQL Transformation
Add SQL transformation and name it `Calculate_capacity_utilization`.  
Reference: https://learn.microsoft.com/fabric/real-time-hub/sql-operator

### Sample SQL Script
```
WITH u AS (
  SELECT
    data.capacityId AS capacityId,
    data.capacityName AS capacityName,
    data.capacitySku AS capacitySku,
    data.windowStartTime AS windowStartTime,
    data.windowEndTime AS windowEndTime,
    CAST(data.baseCapacityUnits AS float) AS baseCapacityUnits,
    CAST(data.capacityUnitMs AS float) AS capacityUnitMs,
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

### Rule Configuration in Activator
- Create object: **Fabric Capacity**
- Unique Identifier: `capacityName`
- Properties: `utilization`
- Create rule for utilization
- Summarization: **Average over 10 minutes**
- Condition: utilization > 0.8 for 10 minutes
- Action: execute notebook `scale_up_by_api`

## Using Fabric REST API to Scale Up SKU
A sample notebook using SP secret from Key Vault is included. (Full code preserved as-is.)

```
# Fabric Notebook - Scale Fabric capacity via ARM using SP secret from Key Vault
import os
import time
import requests
from notebookutils import mssparkutils
from azure.identity import ClientSecretCredential
...
```

Full notebook code can be appended here if needed.
