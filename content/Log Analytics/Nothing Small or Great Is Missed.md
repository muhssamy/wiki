---
tags:
  - PowerBI
  - LogAnalytics
  - Troubleshooting
  - Fabric
created: 2026-01-13
description: When Power BI Desktop started killing our system with mysterious queries, Log Analytics revealed what dashboards couldn't. A technical investigation into the November release bug that almost cost us thousands in wasted capacity upgrades.
cover: Log Analytics/attachments/logs.png
image: Log Analytics/attachments/logs.png
author: Muhammad Samy
---
# Nothing Small or Great Is Missed

![[Log Analytics/attachments/logs.png]]

## Lessons from Power BI Log Analytics

> [!abstract] At an enterprise scale, Power BI log analytics is not optional. It is the only reliable way to understand what is _actually happening_ inside your models — especially when refreshes don't fail, but don't finish either.

---

## The Incident That Triggered Everything

One morning, I received multiple **Data Quality (DQ) check failure emails**.

These checks were part of an **enterprise-level DQ framework** I had implemented and shared in a previous article. When _all_ DQ checks fail at the same time, it usually indicates one thing:

> [!warning] The dataset refresh has not completed yet.

However, in this case:

- There was **no refresh failure notification**
- No error visible in the Power BI UI
- The scheduled refresh window had already passed

At the same time, business users had started their day:

- Opening reports
- Checking yesterday's sales and performance
- Messaging on Teams asking why numbers were not updated

Waiting was not an option.

---

## Refresh Configuration (Important Context)

This is the **default Power BI scheduled refresh setup** with **incremental refresh enabled**.

> [!note] **Refresh configuration:**
> 
> - Parallelism: **6**
> - Retry count: **5**
> - Transactional commit: **Enabled**
> - Large tables using **incremental refresh**

This configuration is generally stable and reliable — **until something downstream starts struggling**.

With this setup, Power BI will:

- Retry refresh operations **silently**
- Avoid **partial commits** to the model
- Delay surfacing errors **until all retries are exhausted**

> [!warning] This means you can have a refresh that is effectively failing **without knowing it yet**.

> [!danger] **The Hidden Cost of Silent Retries**
> 
> Each retry consumes:
> 
> - Fabric capacity resources (CPU, memory)
> - Source system resources (database connections, query execution time) which is Fabric also but another capacity.
> - Time that could be used for other dataset refreshes
> 
> With 5 retry attempts configured, a single failing refresh can consume resources equivalent to **6 full refresh operations** before anyone even knows there's a problem. In a multi-tenant Premium capacity environment, this waste impacts other workloads and datasets.

---

## When the Refresh Did Not Finish

The refresh was expected to complete in approximately **15 minutes**.

After **more than an hour**, it was still running.

There was no failure, no success, and no clear signal from the UI. At this point, waiting for retry exhaustion or user escalation was not acceptable.

> [!tip] When Power BI is silent, logs are not.

---

## Going Where the Truth Lives: Logs

My first stop was **Kusto Explorer**.

I needed to understand **what was happening right now** inside the dataset refresh process — not after it failed, but while it was still running.

This is one of the queries I commonly use to analyze refresh behavior:

```kql
// Find refresh errors and progress signals

PowerBIDatasetsWorkspace
| where TimeGenerated > ago(1h)
| where ArtifactId == '0000-0000-0000-0000' // dataset_id
| where OperationName in ('Error', 'ProgressReportEnd', 'ProgressReportError')
| project XmlaRequestId, TimeGenerated, OperationName, OperationDetailName,
          DurationMs, CpuTimeMs, EventText, Status, StatusCode
| order by TimeGenerated desc
```

Logs do not guess. They do not assume. They record.

---

## What the Logs Revealed

The dataset was:

- Not idle
- Not frozen
- Actively retrying

By isolating one retry attempt, I discovered that:

- A single view in the source system was taking an excessive amount of time
- The SQL execution itself was slow before Power BI even started tabular processing

> [!note] During a refresh, Power BI:
> 
> 1. Executes SQL on the source system
> 2. Reads the data
> 3. Performs tabular processing (segment compression, dictionary encoding, hierarchy building)
> 
> If step one struggles, everything downstream suffers.

---

## Acting Before the Next Retry Failed

Because this behavior was visible during the retry cycle, I could act early.

I immediately worked with our DBA team to:

- Identify the problematic query
- Resolve the source performance issue

The next retry succeeded, and the refresh completed without a visible failure to business users.

Later, we improved the design permanently:

- We stopped relying on the complex view
- Created a physical table in the source system
- Ingested from that table instead

> [!tip] Power BI should select and read data, not execute complex SQL logic during refresh.

---

## Why Logging Matters to Me (Beyond Technology)

Logging is not just a technical mechanism for me.

In Islam, accountability and record-keeping are fundamental concepts. Every action — small or great — is recorded with precision.

The Qur'an describes this clearly:

> [!quote] **"وَوُضِعَ ٱلْكِتَـٰبُ فَتَرَى ٱلْمُجْرِمِينَ مُشْفِقِينَ مِمَّا فِيهِ وَيَقُولُونَ يَـٰوَيْلَتَنَا مَالِ هَـٰذَا ٱلْكِتَـٰبِ لَا يُغَادِرُ صَغِيرَةًۭ وَلَا كَبِيرَةً إِلَّآ أَحْصَىٰهَا ۚ وَوَجَدُوا۟ مَا عَمِلُوا۟ حَاضِرًۭا ۗ وَلَا يَظْلِمُ رَبُّكَ أَحَدًۭا"**
> 
> _"And the record [of deeds] will be placed [open], and you will see the criminals fearful of that within it, and they will say, 'Oh, woe to us! What is this book that leaves nothing small or great except that it has enumerated it?' And they will find what they did present [before them]. And your Lord does injustice to no one."_
> 
> — Surah Al-Kahf (18:49)

This principle — that nothing is ignored, nothing is lost, and everything can be revisited — strongly shapes how I think about system design and data engineering.

---

## From Belief to Engineering Practice

In analytics platforms:

- Logs are the single source of truth
- They reveal what actually happened, not what we assume happened
- They enable accountability, fairness, and learning

A system without logs is blind. A team without logs is reactive.

---

## Final Thought

Dashboards show outcomes. Logs show responsibility.

A refresh may fail silently, but the logs have already written the story.

> [!success] The difference between reacting late and acting early is knowing where to look — and how to read the record.