## Auto-scale Fabric capacity based on utilization

Organizations often face unpredictable, spiky analytics workloads that can lead to throttling and degraded performance when capacity becomes saturated. Since Microsoft Fabric does not provide built‑in autoscale functionality today, this demo introduces an approach to dynamically scale capacity on demand—ensuring consistent performance while controlling costs and avoiding permanent over‑provisioning.

The sample demonstrates how to monitor real‑time capacity usage and automatically trigger a scale‑up action when utilization exceeds defined thresholds. It also calls a notebook to execute additional logic. The accompanying diagram illustrates how the solution ingests capacity signals, processes them, and launches a notebook via Activator within an Eventstream in Real‑Time Intelligence (RTI).

The concept is to calculate utilization using the following formula.

```
utilization = capacityUnitMs / (baseCapacityUnits * windowDurationMs)
```
More info: Explore Capacity overview events.

### Key components

**Capture Capacity Data** – Retrieve capacity metrics from Fabric Capacity Overview events.

**Process Events and Compute Utilization** – Use a SQL operator to process incoming events and calculate capacity utilization.

**Trigger Actions via Activator** – Configure Activator to monitor incoming utilization events and launch a notebook.

### Instruction

**Create an Eventstream** – Set up an event stream.

**Add a Source Connector for Fabric Capacity Overview Events** – Configure event stream to include capacity overview data.

Steps:
- Select Capacity Overview Events as data source and connect.
- Select event type Microsoft.Fabric.Capacity.Summary and choose capacity.
- Rename data source to ‘Fabric-capacity-events’.
- Add SQL transformation code.

### SQL Code
```
WITH u AS (
 SELECT data.capacityId AS capacityId,
        data.capacityName AS capacityName,
        data.capacitySku AS capacitySku,
        data.windowStartTime AS windowStartTime,
        data.windowEndTime AS windowEndTime,
        CAST(data.baseCapacityUnits AS float) AS baseCapacityUnits,
        CAST(data.capacityUnitMs AS float) AS capacityUnitMs,
        CAST(CAST(data.capacityUnitMs AS float) /
            NULLIF(CAST(data.baseCapacityUnits AS float) *
                   DATEDIFF(millisecond, data.windowStartTime, data.windowEndTime), 0) AS float) AS utilization
 FROM [CapacityScaleupdemo-stream]
)
SELECT u.capacityId,
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

### This script performs the following actions
- Reads capacity telemetry from [CapacityScaleupdemo‑stream].
- Calculates utilization.
- Writes results to [Capacity‑Activator].

### Activator rule
- Create object: Fabric Capacity
- Unique Identifier: capacityName
- Properties: utilization
- Create rule: summarization average 10 minutes, condition > 0.8
- Action: Execute notebook scale_up_by_api

### Sample Notebook
(Notebook code preserved exactly as DOCX)

```
# Fabric Notebook - Scale Fabric capacity via ARM using SP secret from Key Vault
import os
import time
import requests
from notebookutils import mssparkutils
from azure.identity import ClientSecretCredential
...
```
