---
tags:
  - PowerBI
  - LogAnalytics
  - Troubleshooting
  - Fabric
created: 2025-11-30
description: When Power BI Desktop started killing our system with mysterious queries, Log Analytics revealed what dashboards couldn't. A technical investigation into the November release bug that almost cost us thousands in wasted capacity upgrades.
cover: Log Analytics/attachments/ChatGPT.png
image: Log Analytics/attachments/ChatGPT.png
author: Muhammad Samy
---
# Power BI Desktop Is Killing Our System

![[Log Analytics/attachments/ChatGPT.png]]
## The Crisis

"Power BI Desktop is killing our system." 

This was the last issue ticket I opened with the Microsoft support team. After the November release, Power BI Desktop began issuing `EVALUATE COLUMNSTATISTICS()` queries that brought our infrastructure to its knees. A single user connecting to a Power BI shared semantic model could push our capacity utilization beyond 200%.

## The Investigation

The issue was incredibly ambiguous at first. I began my analysis using the Fabric Capacity Metrics app, focusing on the exact timepoint when everything spiraled out of control. What I discovered was alarming: one user was consuming more than 48% of our capacity base.

Digging into Log Analytics to understand what this user was doing, I found they were simply connected from Power BI Desktop, executing the query `EVALUATE COLUMNSTATISTICS()`. This raised immediate questions: How was a user with only Build access running this query? What value were they even getting from these results?

Then it happened again—a different user, same query, same Power BI Desktop connection. And again. And again. It became a nightmare scenario for our team.

## The Resolution

Convincing the Microsoft team that a genuine issue existed took considerable effort. After sharing our Log Analytics data and Power BI Desktop traces, we finally received this response:

> The Engineering team has observed that the queries generated from the November version are noticeably different and somewhat more complex compared to those from the October release.

Thanks to the support team's responsiveness, we received a new build. I tested it thoroughly, and it's now heading to production.

## The Critical Insight

Here's the key takeaway: **without Log Analytics, we would never have identified what was happening at these critical timepoints or where our resources were being wasted.**

In the aftermath of a migration project, high usage is often celebrated as strong user adoption. This can lead to premature decisions about scaling capacity and budget. In our case, doubling the capacity would have been a massive waste—the issue would have persisted regardless. Alhamdulillah, we identified the root cause before making that costly mistake.

## What's Next

In this article, I'll walk you through step-by-step how to investigate incidents like this using Log Analytics, and why I prefer it over the built-in workspace monitoring capabilities.

---

## Why Log Analytics Over Workspace Monitoring?

Workspace monitoring has significant limitations that make it unsuitable for system-wide investigations:

**Resource Consumption**: Workspace monitoring runs as a background task that consumes from our base capacity. During a crisis, the last thing you need is your monitoring solution adding to the problem.

**Fragmented Visibility**: There's no centralized monitoring dashboard. Workspace monitoring must be enabled per workspace, which scatters your logs across multiple locations. When you're troubleshooting a capacity-wide issue affecting multiple workspaces, this fragmentation becomes a critical blind spot.

While workspace monitoring has its benefits for workspace-level insights, it falls short for enterprise-scale Power BI system monitoring and troubleshooting.

---

## Step 1: Start with Fabric Capacity Metrics

The Fabric Capacity Metrics app is your first stop when investigating throttling incidents.

**Here's how to zero in on the problem:**

1. Navigate to the specific timepoint when throttling occurred
2. Zoom into that exact moment to explore what was happening
3. Focus on **Interactive Operations**—most Power BI operations fall into this category
4. Look for anomalies: unusually high CPU usage, extended durations, or suspicious patterns

**Pro tip**: Once you identify a suspicious user or operation, add the **OperationID** from the optional columns as shown below. This ID is your key to deep-diving in Log Analytics.

![[Pasted image 20251130202559.png]]

![[Pasted image 20251130202739.png]]

---

## Step 2: Deep Dive with Log Analytics

Log Analytics provides the granular detail you need to understand exactly what happened.

### Setting Up Your Connection

You can access Log Analytics in two ways:
- **Kusto Explorer**: A dedicated query tool for advanced analysis
- **Azure Portal**: Run queries directly in your Log Analytics workspace

If you haven't linked your Power BI workspace to Log Analytics yet, follow Microsoft's official guide: [Using Azure Log Analytics in Power BI](https://learn.microsoft.com/en-us/power-bi/transform-model/log-analytics/desktop-log-analytics-overview)

### Investigating a Specific Request

Use this query to examine what happened during a specific operation:
```kql
PowerBIDatasetsWorkspace
| where TimeGenerated > ago(1h) //adjust time to match the timepoint from Fabric Capacity Metrics
| where OperationName == "QueryEnd"  
| where XmlaRequestId == '' //paste the OperationID from Fabric Capacity Metrics app
| project ExecutingUser, ApplicationName, ArtifactName, EventText, TimeGenerated, Status, XmlaRequestId, XmlaSessionId, DurationMs
| order by TimeGenerated desc;
```

**What this reveals**: You'll see exactly what Power BI Desktop was generating—in our case, the problematic `EVALUATE COLUMNSTATISTICS()` query that shouldn't have been running at all.

![[Pasted image 20251130202931.png]]
### Checking If You're Affected

Want to know if this issue hit your environment? Run this query:
```kql
PowerBIDatasetsWorkspace
| where TimeGenerated > ago(10h) //adjust timeframe as needed
| where EventText has "EVALUATE COLUMNSTATISTICS()"
| order by TimeGenerated desc;
```

If you see results, you've been impacted. The timestamp will show you when it started, and the frequency will indicate the severity.

---

## The Bigger Picture

This investigation reinforced a critical lesson: **visibility is everything**. Without Log Analytics, we would have been operating on assumptions:
- Usage is up because adoption is high
- We need more capacity
- Users are just running heavy reports

All wrong. All expensive.

The right monitoring tools don't just help you react to problems—they help you understand them, solve them properly, and avoid costly mistakes.

---

*Have you experienced similar capacity issues? What monitoring approach works best in your environment? Let's discuss in the comments.*

**Tags**: #PowerBI #LogAnalytics #FabricCapacity #Monitoring #Troubleshooting #DataEngineering