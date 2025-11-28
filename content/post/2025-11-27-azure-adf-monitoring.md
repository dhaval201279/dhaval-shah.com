---
title: Building bulletproof ADF pipelines
author: Dhaval Shah
type: post
date: 2025-11-27T02:00:50+00:00
url: /building-bulletproof-adf-pipelines/
categories:
  - azure
  - data-engineering
tags:
  - azure
  - data-engineering
thumbnail: "images/wp-content/uploads/2024/12/industrial_pipes.jpg"
---

[![](https://www.dhaval-shah.com/images/wp-content/uploads/2024/12/industrial_pipes.jpg)](https://www.dhaval-shah.com/images/wp-content/uploads/2024/12/industrial_pipes.jpg)
-----------------------------------------------------------------------------------------------------------------------------------------

# Background

As software engineers, we spend a lot of time building data pipelines, but we often overlook what happens when things go wrong. In the world of Azure Data Factory i.e. [ADF] (), a pipeline that fails silently is a nightmare.

If you are just starting out with ADF, embracing industry standards would certainly ease out troubleshooting and monitoring needs. Here is my practical guide to setting up error management, logging and alerting the right way.

So this post will mainly cover:
1. High level overview of what is ADF
2. Error Handling
3. Alerts 

## 1. High level overview of ADF
Azure Data Factory is composed of the following key components:

1. Pipelines
2. Activities
3. Datasets
4. Linked services
5. Data Flows
6. Integration Runtimes

## 2. Error Handling
As a beginner, use a “standard error-handling pattern” in every important pipeline so you can consistently capture failures and their details.

Most common approach is to use:
- Try/Catch with “On Failure” path 
<IMG :: 1-adf-error-handling-try-catch-approach_0>
- Do "If Else" block with success and failure paths
<IMG :: 1-adf-error-handling-approach_1>

## 3. Alert Management
Alerts in Azure world consists of [various types](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-types). Since we need broader control to define conditions on which alerts must be triggered, we would be configuring [Log Search Alerts](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-types#log-alerts). Log search alert mainly comprises o - 
1. Scope - Allows to select resources
2. Alert Conditions - Where we will be adding our custom log search query
3. Actions - Action that platform must take, once Alert conditions are met


Before delving into the details of alerts, first lets understand the use case. Only then can one connect ADF components and their corresponding alerts with the conditions that trigger them.

### 3.1 High level use case
Ask is to collate data from multiple tables withing multiple data sources i.e. CSV files from [Azure Blob Storage]() location, [Azure CosmosDB]() and thereby aggregate / transform them to generate a CSV file that ultimately gets pushed to destination [Azure Blob Storage]() location

<IMG :: >

Considering nature of ADF pipeline, I have split alerts into 2 broad categories:
1. Infrastructure Alerts
2. ADF Alerts

### 3.2 Infrastructure Alerts
At an infrastructure level, ADF pipeline is dependent on CosmosDB and Azure Blob Storage. Hence we have setup alerts for:

#### 3.2.1 Azure Blob Storage Alerts

| Alert Description   | Log search query for Alert Conditions     | Severity   |
| --------  | -------- | :-----: |
| Alert for non-existence of source folder | `StorageBlobLogs \| where tostring(ObjectKey) contains_cs "blob-strg-ac/src-folder" \| where StatusText == "PathNotFound"` | Critical |
| Alert for non-existence of destination folder | `StorageBlobLogs \| where tostring(ObjectKey) contains_cs "blob-strg-ac-dest/output-folder" \| where StatusText == "PathNotFound"` | Critical |
| Alert for non-existence of generated CSV file | `ADFActivityRun \| where Status == "Failed" \| order by TimeGenerated desc` | Critical |

#### 3.2.2 Azure ComsosDB Alerts

| Alert Description   | Log search query for Alert Conditions     | Severity   |
| --------  | -------- | :-----: |
| Alert for CosmosDB Failures | `ADFActivityRun \| where Status == "Failed" \| where Error has "cosmosDB" \| order by TimeGenerated desc` | Critical |
| Alert for CosmosDB Throttling (i.e. 429 Errors) | `AzureDiagnostics \| where Category == "DataPlaneRequests" \| where statusCode_s == "429" \| order by TimeGenerated desc` | Critical |
| Alert for CosmosDB Query latency (> 10 sec) | `AzureDiagnostics \| where OperationName == "Query" \| where databaseName_s == "cosmos-db-container-name" \| extend DurationMsNumeric = todouble(duration_s) \| summarize AvgLatencyMs = avg(DurationMsNumeric), RequestCount = count(), MaxLatencyMs = max(DurationMsNumeric), MinLatencyMs = min(DurationMsNumeric) by bin(TimeGenerated, 1h), Resource, ResourceGroup, SubscriptionId, OperationName \| where AvgLatencyMs > 10 \| order by TimeGenerated desc` | Critical |

### 3.3 ADF Alerts
Since ADF consists of multiple components, we need to create alerts for failures w.r.t:
1. Pipeline
2. Activity
3. Integration Runtime and Data Movements

#### 3.3.1 Pipeline Alerts

| Alert Description   | Log search query for Alert Conditions     | Severity   |
| --------  | -------- | :-----: |
| Alert for Pipeline Failures | `PipelineFailedRuns > 0` | Critical |
| Alert for Pipeline Execution Duration | [Ref 1] | Critical |

**Ref 1**
``` kql
let InProgressRuns = 
    ADFPipelineRun
    | where Status == "InProgress"
    | order by TimeGenerated desc;

let CompletedRuns = 
    ADFPipelineRun
    | where Status in ("Succeeded", "Failed", "Cancelled")
    | order by TimeGenerated desc;

InProgressRuns
| join kind=leftanti (CompletedRuns) on RunId
| extend RunDuration = now() - Start
| where RunDuration > 24h and TimeGenerated >= ago(30d)
```

#### 3.3.2 Activity Alerts

| Alert Description   | Log search query for Alert Conditions     | Severity   |
| --------  | -------- | :-----: |
| Alert for Activity Failures | `ActivityFailedRuns > 0` | Critical |
| Alert for Activity Execution Duration | [Ref 2] | Critical |

**Ref 2**
``` kql
let InProgressRuns = 
    ADFActivityRun
    | where Status == "InProgress"
    | order by TimeGenerated desc;

let CompletedRuns = 
    ADFActivityRun
    | where Status in ("Succeeded", "Failed", "Cancelled")
    | order by TimeGenerated desc;

InProgressRuns
| join kind=leftanti (CompletedRuns) on ActivityRunId
| extend RunDuration = now() - Start
| where RunDuration > 24h and TimeGenerated >= ago(30d)
```

#### 3.3.3 Integration Runtime and Data Movement Alerts

| Alert Description   | Log search query for Alert Conditions     | Severity   |
| --------  | -------- | :-----: |
| Alert for Integration Runtime availability | `IntegrationRuntimeAvailableNodeNumber < 1` | Critical |
| Alert for Blob Storage Read failures | [Ref 4] | Critical |
| Alert for Blob Storage Write failures | [Ref 5] | Critical |

**Ref 4**
``` kql
StorageBlobLogs
| where ObjectKey has "strg-blob-container-name/src-folder"
| where Category == 'StorageRead'
| where StatusText != 'Success'
```

**Ref 5**
``` kql
ADFActivityRun 
| where ActivityType == "Copy"
| where Status == "Failed"
```
# Wrapping Up: Building a Resilient ADF Pipeline

We've walked through the essentials of setting up **bulletproof error handling and alerts** for your ADF pipeline - the one that brings data together from all sources and lands it safely in Azure Blob Storage. The takeaway? When your production pipelines are backed by this level of maturity, troubleshooting stops being a nightmare and starts being a precise & quick. Your data workflow is now **resilient and reliable.**