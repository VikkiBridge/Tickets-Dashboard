# 📊 Tickets-Dashboard
---
## 🏢 Business Problem & Objective
The internal IT Support team was experiencing a high volume of support tickets across multiple functions and branches, but lacked a centralised, real-time mechanism to monitor performance and service-level agreements (SLAs).
--- 
## **Key Pain Points:**
* Ticket backlogs, aging unassigned incidents and breach risks were hidden inside raw system logs, requiring manual report generation to review.
* Team leads could not easily identify sytemic infrastructure issues (e.g., a sudden spike in login failures at a specific branch) versus iaolated user requests.
* Management had no clear visibility into peak operational hours or individual technician utilisation, leading to imbalanced workloads and delayed resolution times.
---
## **The Objective**
The objective of thisproject was to develop a dynamic, operational Power BI dashboard connected directly to the HaloITSM data source.  The report serves as a "single pane of glass" for the IT Support Manager and Team Leads to track real-time queue health, optimise technician workflows, and protect business continuity.
## **Target Audience & Core KPIs**
* **Primary Users:** IT Support Desk Managers, Infrastructure Leads, and Operations Directors.
* **Core Metrics Tracked:**
  * **SLA Compliance %:** Percentage of tickets responded to and resolved within agreed corporate timeframes.
  * **Ticket Velocity & Backlog:** Net-new incoming tickets versus daily closure rates to track queue health.
  * **Average Resolution Time (ART):** The mean time taken from ticket creation to final closure, segmented by priority (P1-P4).
  * **Aging Backlog:** Highlighting tickets left open for 0-7, 8-14, or 14 or more days to prevent stale issues.      
---
## 🛠️ Tech Stack & Architecture
* **Tool:** Power BI Desktop / Fabric Cloud
* **Data Sources:** [e.g., SQL Server, SharePoint Lists, API JSON extracts]
* **Data Model:** Star Schema (1:Many relationships)
---
## 📐 Data Modeling & DAX Highlights
Engineer Comparison Table = 
VAR BaseTable = 'DimMetricNames'
RETURN
SELECTCOLUMNS(
    BaseTable,
    "Metric Name", 'DimMetricNames'[Metric Name],
    "Apprentice Value", 
        VAR AppRaw =
        SWITCH(
            'DimMetricNames'[Metric Name],
            "Categories Handled", CALCULATE([Unique Categories Count], DimAgent[Job Role] = "Apprentice"),
            "Tickets Closed", CALCULATE([Tickets Closed], 'DimAgent'[Job Role] = "Apprentice"),
            "FCR Rate", CALCULATE([FCR Rate %], 'DimAgent'[Job Role] = "Apprentice"),
            "Avg Resolution (days)", CALCULATE([Avg Resolution (days)], 'DimAgent'[Job Role] = "Apprentice"),
            BLANK()
        )
        RETURN
        SWITCH(
            'DimMetricNames'[Metric Name],
            "Categories Handled", FORMAT(AppRaw, "#,##0"),
            "Tickets Closed", FORMAT(AppRaw, "#,##0"),
            "FCR Rate", FORMAT(AppRaw, "0.0%"),
            "Avg Resolution (days)", FORMAT(AppRaw, "0.00")
        ),
    "Senior Value",
        VAR SenRaw = 
        SWITCH(
            'DimMetricNames'[Metric Name],
            "Categories Handled", CALCULATE([Unique Categories Count], DimAgent[Job Role] = "Senior"),
            "Tickets Closed", CALCULATE([Tickets Closed], 'DimAgent'[Job Role] = "Senior"),
            "FCR Rate", CALCULATE([FCR Rate %], 'DimAgent'[Job Role] = "Senior"),
            "Avg Resolution (days)", CALCULATE([Avg Resolution (days)], 'DimAgent'[Job Role] = "Senior"),
            BLANK()
        )
         RETURN
        SWITCH(
            'DimMetricNames'[Metric Name],
            "Categories Handled", FORMAT(SenRaw, "#,##0"),
            "Tickets Closed", FORMAT(SenRaw, "#,##0"),
            "FCR Rate", FORMAT(SenRaw, "0.0%"),
            "Avg Resolution (days)", FORMAT(SenRaw, "0.00")
),
    "Gap",
    VAR AppVal = 
        SWITCH(
            'DimMetricNames'[Metric Name],
            "Categories Handled", CALCULATE([Unique Categories Count], DimAgent[Job Role] = "Apprentice"),
            "Tickets Closed", CALCULATE([Tickets Closed], 'DimAgent'[Job Role] = "Apprentice"),
            "FCR Rate", CALCULATE([FCR Rate %], 'DimAgent'[Job Role] = "Apprentice"),
            "Avg Resolution (days)", CALCULATE([Avg Resolution (days)], 'DimAgent'[Job Role] = "Apprentice"),
            BLANK()
        )
    VAR SenVal = 
        SWITCH(
            'DimMetricNames'[Metric Name],
            "Categories Handled", CALCULATE([Unique Categories Count], DimAgent[Job Role] = "Senior"),
            "Tickets Closed", CALCULATE([Tickets Closed], 'DimAgent'[Job Role] = "Senior"),
            "FCR Rate", CALCULATE([FCR Rate %], 'DimAgent'[Job Role] = "Senior"),
            "Avg Resolution (days)", CALCULATE([Avg Resolution (days)], 'DimAgent'[Job Role] = "Senior"),
            BLANK()
        )
   -- Logic: Rates use Point Difference, others use Percentage Variance
   VAR IsRate = CONTAINSSTRING('DimMetricNames'[Metric Name], "Rate")
   VAR RawGap = IF(IsRate, (AppVal - SenVal) * 100, DIVIDE(AppVal - SenVal, SenVal, 0))
   RETURN
       SWITCH( TRUE(),
           IsRate, FORMAT(RawGap, "+0.0 pts;-0.0 pts;0 pts"),
           FORMAT(RawGap, "+0%;-0%;0%")
       ),
"Trend",
   VAR MetricName = 'DimMetricNames'[Metric Name]
   -- Calculate Gap for Current and Prior period
   VAR CurrGap = [Metric Gap %]
   VAR PrevGap = CALCULATE([Metric Gap %], DATEADD('DimCalendar'[Date], -1, MONTH))
   VAR Improvement = CurrGap - PrevGap
   RETURN
       SWITCH( TRUE(),
           -- For Resolution Days, a DECREASING gap is an improvement
           MetricName = "Avg Resolution (days)",
               IF(Improvement < 0, "↑ Improving", "↓ Declining"),
           -- For Reopened Rate, a LOWER rate is better (so a decreasing gap is good)
           MetricName = "Reopened Rate",
               IF(Improvement < 0, "↑ Improving", "↓ Declining"),
           -- For everything else (Closed, FCR, Categories), HIGHER is better
           IF(Improvement > 0, "↑ Improving", "↓ Declining")
       )
)
