---
title: Building Bulletproof ADF Pipelines
author: Dhaval Shah
type: post
date: 2025-11-27T02:00:50+00:00
url: /building-bulletproof-adf-pipelines/
categories:
  - azure
  - ADF
  - data-engineering
tags:
  - azure
  - data-engineering
  - ADF
thumbnail: "images/wp-content/uploads/2025/11/thumbnail-pipeline-2.jpg"
---

[![](https://www.dhaval-shah.com/images/wp-content/uploads/2025/11/thumbnail-pipeline-2.jpg)](https://www.dhaval-shah.com/images/wp-content/uploads/2025/11/thumbnail-pipeline-2.jpg)
-----------------------------------------------------------------------------------------------------------------------------------------

# Background

As software engineers we spend so much of our time building these beautiful, complex data pipelines. We design them to move mountains of information... but let's be honest, we often cross our fingers and hope they don't break. And when they do? A pipeline that fails without making a peep - especially in a system like [Azure Data Factory (ADF)](https://learn.microsoft.com/en-us/azure/data-factory/introduction) - is a genuine nightmare.

Working with Azure Data Factory (ADF), I’ve learned that silent failures are the kind you want to avoid at all costs. If you’re new to ADF, adopting some good practices early on makes troubleshooting and monitoring far less painful.

That’s why I’ve put together this practical guide. It’s not theory or textbook stuff—it’s the lessons I’ve picked up while setting up error handling, logging, and alerts in real projects.

Here’s what I’ll walk you through:
1. A cursory overview of what Azure Data Factory (ADF) actually is
2. The use case we're implementing
3. Error Handling: Building a Safety Net for Your Data
4. Setting up effective Alerts.

# 1. A cursory overview of what Azure Data Factory (ADF) actually is
To effectively implement ADF pipelines, you need to know its basic structure. I like to think of it like a coordinated job site. Each part has a specific role, and they all work together to move the data:

1. **Pipelines**: The overall Project Plan — the sequence of steps we need to execute.
2. **Activities**: The individual Workers on the site who have specific tasks like drilling, moving, or mixing.
3. **Datasets**: What data looks like at the source and the sink.
4. **Linked Services**: The Credentials and Keys needed to access the storage, databases, or external systems.
5. **Data Flows**: The Specialized Equipments for heavy duty - Visual data transformation (when you need to reshape data without writing boiler plate code).
6. **Integration Runtimes**: The actual Machinery and Power Source that runs everything — the compute environment.

# 2. The use case we're implementing

[![Use Case](https://www.dhaval-shah.com/images/wp-content/uploads/2025/11/azure-adf-pipeline-architecture.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2025/11/azure-adf-pipeline-architecture.png)

We're tackling a classic real-world problem: <u>**Data Sprawl**</u>

Here is what we need to accomplish: 
- We have critical business data living in two different neighborhoods
- Standardized data sitting in a bunch of CSV files inside our cloud storage bucket i.e. [Azure Blob Storage](https://learn.microsoft.com/en-us/azure/storage/blobs/)
- Semi-structured data stored in [Azure Cosmos DB](https://learn.microsoft.com/en-us/azure/cosmos-db/) .

We need to build a pipeline that goes out, collects all those pieces, then performs some necessary calculations and clean-up (transform/aggregate). The end goal is to create one single, easy-to-read CSV file and deliver it straight to another dedicated folder in our Azure Blob Storage.

This is a perfect example of what ADF is designed to do - seamlessly knit together data from multiple homes.

# 3. Error Handling: Building a Safety Net for Your Data
When you're just starting out, the best piece of advice I can give you is this: **don't skip the safety net**. Make sure you adopt a standard **error-handling** pattern and use it religiously in every critical pipeline you build. This consistency is what allows you to reliably capture failures and all their important details - it's the foundation of troubleshooting!

The goal is to make sure your pipeline doesn't just crash silently; it needs to report back exactly what went wrong.

The two most common and reliable methods for achieving this are:
1. **Try/Catch with “On Failure” path**

[![Try/Catch with On Failure](https://www.dhaval-shah.com/images/wp-content/uploads/2025/11/1-adf-error-handling-try-catch-approach_0.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2025/11/1-adf-error-handling-try-catch-approach_0.png)

2. **Do "If Else" block with success and failure paths**

[![If Else block with Success and Failure](https://www.dhaval-shah.com/images/wp-content/uploads/2025/11/1-adf-error-handling-approach_1.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2025/11/1-adf-error-handling-approach_1.png)

# 4. Setting up effective Alerts
Okay, we've built the safety net (Error Handling), but now we need the alarm system! A silent failure is the enemy, so setting up effective alerts is critical.

In the Azure universe, there are several ways to configure Alerts, but we need something that gives us broader control for configuring and setting up ADF pipeline alerts. That's why we're going straight for Log Search Alerts. They are powerful because they let us define custom criteria based on the logs generated by our system.

A Log Search Alert essentially has three parts:
- **Scope**: This is where we select the specific resources that the alert should watch.
- **Alert Conditions**: We write a custom search query that defines the exact scenario that must happen to trigger the alert.
- **Actions**: What should happen when the alert fires? Send an email? Hit a webhook?

Because our ADF pipeline relies on many moving pieces, I like to split our alerting strategy into two clear buckets:
1. **Infrastructure Alerts** (_What's happening around ADF?_)
2. **ADF Alerts** (_What's happening inside ADF?_)

## 4.1 Infrastructure Alerts
Our pipeline depends entirely on our external services behaving well. If our dependencies — like the storage or the database are struggling, our pipeline will fail, too. So, we set up alerts for:

### 4.1.1 Azure Blob Storage Alerts

| Alert Description   | Log search query for Alert Conditions     | Severity   |
| --------  | -------- | :-----: |
| Alert for non-existence of source folder | <code>StorageBlobLogs \| where tostring(ObjectKey) contains_cs "blob-strg-ac/src-folder" \| where StatusText == "PathNotFound"</code> | Critical |
| Alert for non-existence of destination folder | <code>StorageBlobLogs \| where tostring(ObjectKey) contains_cs "blob-strg-ac-dest/output-folder" \| where StatusText == "PathNotFound"</code> | Critical |
| Alert for non-existence of generated CSV file | <code>ADFActivityRun \| where Status == "Failed" \| order by TimeGenerated desc</code> | Critical |

### 4.1.2 Azure ComsosDB Alerts

| Alert Description   | Log search query for Alert Conditions     | Severity   |
| --------  | -------- | :-----: |
| Alert for CosmosDB Failures | <code>ADFActivityRun \| where Status == "Failed" \| where Error has "cosmosDB" \| order by TimeGenerated desc</code> | Critical |
| Alert for CosmosDB Throttling (i.e. 429 Errors) | <code>AzureDiagnostics \| where Category == "DataPlaneRequests" \| where statusCode_s == "429" \| order by TimeGenerated desc</code> | Critical |
| Alert for CosmosDB Query latency (> 10 sec) | Ref 0 | Critical |

**Ref 0**
```kusto
AzureDiagnostics 
    | where OperationName == "Query" 
    | where databaseName_s == "cosmos-db-container-name" 
    | extend DurationMsNumeric = todouble(duration_s) 
    | summarize AvgLatencyMs = avg(DurationMsNumeric), RequestCount = count(), MaxLatencyMs = max(DurationMsNumeric), MinLatencyMs = min(DurationMsNumeric) by bin(TimeGenerated, 1h), Resource, ResourceGroup, SubscriptionId, OperationName 
    | where AvgLatencyMs > 10 
    | order by TimeGenerated desc
```

## 4.2 ADF Alerts
This is where we check the actual data movement engine. Since ADF is made of:
1. Pipeline
2. Activity
3. Integration Runtime and Data Movements

we need specific alerts for each failure point:

### 4.2.1 Pipeline Alerts

| Alert Description   | Log search query for Alert Conditions     | Severity   |
| --------  | -------- | :-----: |
| Alert for Pipeline Failures | <code>PipelineFailedRuns > 0</code> | Critical |
| Alert for Pipeline Execution Duration | [Ref 1] | Critical |

**Ref 1**
```kusto
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
  | where RunDuration > 1h 
      and TimeGenerated >= ago(1d)
```

### 4.2.2 Activity Alerts

| Alert Description   | Log search query for Alert Conditions     | Severity   |
| --------  | -------- | :-----: |
| Alert for Activity Failures | <code>ActivityFailedRuns > 0</code> | Critical |
| Alert for Activity Execution Duration | [Ref 2] | Critical |

**Ref 2**
```kusto
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
  | where RunDuration > 1h and TimeGenerated >= ago(1d)
```

### 4.2.3 Integration Runtime and Data Movement Alerts

| Alert Description   | Log search query for Alert Conditions     | Severity   |
| --------  | -------- | :-----: |
| Alert for Integration Runtime availability | <code>IntegrationRuntimeAvailableNodeNumber < 1</code> | Critical |
| Alert for Blob Storage Read failures | [Ref 3] | Critical |
| Alert for Blob Storage Write failures | [Ref 4] | Critical |

**Ref 3**
```kusto
StorageBlobLogs
  | where ObjectKey has "strg-blob-container-name/src-folder"
  | where Category == 'StorageRead'
  | where StatusText != 'Success'
```

**Ref 4**
```kusto
ADFActivityRun 
  | where ActivityType == "Copy"
  | where Status == "Failed"
```

# Wrapping Up: Building a Resilient ADF Pipeline

We've walked through the essentials of setting up **bulletproof error handling and alerts** for your ADF pipeline - the one that brings data together from all sources and lands it safely in Azure Blob Storage. The takeaway? When your production pipelines are backed by this level of maturity, troubleshooting stops being a nightmare and starts being a precise & quick. Your data workflow is now **resilient and reliable.**

We walked through the essentials of setting up a **bulletproof error handling framework and smart / loud alerts** for your Azure Data Factory pipeline — the one that brings data together from all its different homes and lands it safely.

The most important takeaway?

When your production pipelines are backed by this level of operational maturity, troubleshooting stops being a late-night nightmare and starts being precise. You can finally trust your data workflow to be resilient and truly reliable.