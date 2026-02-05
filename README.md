

let lookback = 24h;
let base =
requests
| where timestamp > ago(lookback)
| where client_Type != "Browser"
| extend HttpMethod =
    case(
        name startswith "GET ",  "GET",
        name startswith "POST ", "POST",
        name startswith "PUT ",  "PUT",
        name startswith "DELETE ","DELETE",
        name startswith "PATCH ","PATCH",
        "OTHER"
    )
| extend Endpoint = trim_start(@"GET |POST |PUT |DELETE |PATCH ", name)
| extend StatusCode = toint(resultCode)
| summarize
    TotalSamples = count(),
    TotalPassed  = countif(success == true),
    TotalFailed  = countif(success == false),
    Count4xx     = countif(StatusCode between (400 .. 499)),
    Count5xx     = countif(StatusCode between (500 .. 599))
  by HttpMethod, Endpoint
| extend FailureRatePct = round(100.0 * todouble(TotalFailed) / todouble(TotalSamples), 2);

let totals =
requests
| where timestamp > ago(lookback)
| where client_Type != "Browser"
| extend StatusCode = toint(resultCode)
| summarize
    TotalSamples = count(),
    TotalPassed  = countif(success == true),
    TotalFailed  = countif(success == false),
    Count4xx     = countif(StatusCode between (400 .. 499)),
    Count5xx     = countif(StatusCode between (500 .. 599))
| extend HttpMethod = "ALL", Endpoint = "ALL ENDPOINTS",
         FailureRatePct = round(100.0 * todouble(TotalFailed) / todouble(TotalSamples), 2);

base
| union totals
| order by (Endpoint == "ALL ENDPOINTS") asc, FailureRatePct desc, TotalSamples desc


















<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" style="height:64px;margin-right:32px"/>

## 1. Task Restatement

You want to expand the endpoint summary table with additional charts/tables for: **failure analysis, throughput analysis, response time per operation, and Top 10 slow SQL/Snowflake/API calls**.[^1][^2][^3]

## 2. Assumptions (if any)

- Your Application Insights already tracks dependencies (SQL, Snowflake, HTTP) with `dependencies` table.[^2][^1]
- Snowflake calls are logged as `type="Snowflake"` or `type="HTTP"` with target containing "snowflake" (adjust if different).[^1]
- You want a **single comprehensive workbook** with multiple tabs/sections.[^3][^4]


## 3. Plan / Solution (ordered bullets)

I'll give you **12 additional panels** organized into 4 sections: **Failure Analysis (4 panels), Throughput Analysis (2 panels), Response Time Analysis (3 panels), Top Slow Calls (3 panels)**.[^2][^3]

***

## Complete Dashboard Structure (Workbook Layout)

### Section 1: Endpoint Summary (you already have this)

**Panel 1: Endpoint Summary Table** *(from previous response)*

***

### Section 2: FAILURE ANALYSIS (4 panels)

#### Panel 2A: Failure Trend (Time Series - Errors Over Time)

**Purpose**: Spot when failures started and correlate with deployments.[^5][^3]

**Visualization**: Line chart (time series)

**KQL Query**:

```kql
let lookback = 24h;
requests
| where timestamp > ago(lookback)
| summarize 
    TotalRequests = count(),
    Failed4xx = countif(toint(resultCode) between (400 .. 499)),
    Failed5xx = countif(toint(resultCode) between (500 .. 599))
    by bin(timestamp, 5m)
| render timechart 
```

**Chart Settings**:[^3][^5]

- Visualization: **Line chart**
- Y-axis: Stack the series (TotalRequests as area, Failed4xx/5xx as lines on top)
- Add **deployment annotations** (use workbook time brush to correlate with release times)

***

#### Panel 2B: Top 10 Failing Endpoints (by Error Count)

**Purpose**: Focus on which endpoints contribute most to total failure budget.[^6][^2]

**Visualization**: Horizontal bar chart

**KQL Query**:

```kql
let lookback = 24h;
requests
| where timestamp > ago(lookback) and success == false
| summarize 
    FailureCount = count(),
    Failed4xx = countif(toint(resultCode) between (400 .. 499)),
    Failed5xx = countif(toint(resultCode) between (500 .. 599))
    by name
| order by FailureCount desc
| take 10
```


***

#### Panel 2C: Exception Breakdown (Exception Types \& Counts)

**Purpose**: Understand *what* exceptions are causing failures (NullRef, Timeout, SQL, etc).[^7][^8]

**Visualization**: Table

**KQL Query**:

```kql
let lookback = 24h;
exceptions
| where timestamp > ago(lookback)
| summarize 
    ExceptionCount = count(),
    AffectedOperations = dcount(operation_Name),
    SampleMessage = take_any(outerMessage)
    by type
| order by ExceptionCount desc
| take 15
```

**Optional Enhancement**: If you log custom exception metadata in `customDimensions`, parse it:[^8][^7]

```kql
exceptions
| extend ErrorCode = tostring(customDimensions["ErrorCode"]),
         CustomerId = tostring(customDimensions["CustomerId"])
| summarize count() by ErrorCode, type
```


***

#### Panel 2D: Failed Requests with Correlated Exceptions (Root Cause Drill-Down)

**Purpose**: Link failed API calls to the exceptions that caused them (via `operation_Id`).[^9]

**Visualization**: Table

**KQL Query**:

```kql
let lookback = 1h;
requests
| where timestamp > ago(lookback) and resultCode >= "500"
| join kind=inner (
    exceptions
    | where timestamp > ago(lookback)
    | project operation_Id, ExceptionType = type, ExceptionMessage = outerMessage
) on operation_Id
| project 
    timestamp, 
    Endpoint = name, 
    ResultCode = resultCode, 
    Duration = duration,
    ExceptionType, 
    ExceptionMessage,
    operation_Id
| order by timestamp desc
| take 50
```

**Usage**: Click `operation_Id` → go to End-to-end transaction details to trace full request.[^9]

***

### Section 3: THROUGHPUT ANALYSIS (2 panels)

#### Panel 3A: Request Rate Trend (Requests Per Minute)

**Purpose**: Detect traffic spikes, drops, or growth trends.[^2][^3]

**Visualization**: Area chart

**KQL Query**:

```kql
let lookback = 24h;
requests
| where timestamp > ago(lookback)
| make-series RequestsPerMin = count() default=0 on timestamp from ago(lookback) to now() step 1m
| render timechart 
```

**Chart Settings**:[^5][^3]

- Use `make-series` so missing time buckets show as 0 (not gaps)
- Add a **dynamic grain** using workbook time parameter `{TimeRange:grain}` so it auto-adjusts bin size

**Advanced**: Segment by endpoint to see which endpoint drives traffic:

```kql
requests
| where timestamp > ago(24h)
| make-series RequestsPerMin = count() default=0 on timestamp from ago(24h) to now() step 5m by name
| render timechart 
```


***

#### Panel 3B: Top 10 Endpoints by Request Volume

**Purpose**: Identify hottest endpoints for capacity planning.[^2]

**Visualization**: Horizontal bar chart

**KQL Query**:

```kql
let lookback = 24h;
requests
| where timestamp > ago(lookback)
| summarize 
    TotalRequests = count(),
    RequestsPerMin = count() / (24.0 * 60)
    by name
| order by TotalRequests desc
| take 10
```


***

### Section 4: RESPONSE TIME ANALYSIS (3 panels)

#### Panel 4A: Response Time Percentiles Over Time (p50, p95, p99)

**Purpose**: Track latency degradation; p95/p99 catch tail latency.[^3][^2]

**Visualization**: Line chart

**KQL Query**:

```kql
let lookback = 24h;
requests
| where timestamp > ago(lookback)
| summarize 
    P50 = percentile(duration, 50),
    P95 = percentile(duration, 95),
    P99 = percentile(duration, 99)
    by bin(timestamp, 5m)
| render timechart 
```

**Chart Settings**:[^3][^2]

- Y-axis label: "Duration (ms)"
- Add horizontal reference lines at SLO thresholds (e.g., 3000ms for p95)

***

#### Panel 4B: Response Time by Endpoint (Table)

**Purpose**: Compare latency across endpoints; identify slow ones.[^10][^2]

**Visualization**: Table with conditional formatting

**KQL Query**:

```kql
let lookback = 24h;
requests
| where timestamp > ago(lookback)
| summarize 
    RequestCount = count(),
    AvgDuration = round(avg(duration), 2),
    P50 = round(percentile(duration, 50), 2),
    P95 = round(percentile(duration, 95), 2),
    P99 = round(percentile(duration, 99), 2)
    by Endpoint = name
| extend HealthStatus = case(
    P95 > 5000, "Critical",
    P95 > 3000, "Warning",
    "Healthy"
)
| order by P95 desc
```

**Workbook Settings**:[^11][^4]

- Add **conditional formatting**: Color `HealthStatus` column (Red=Critical, Yellow=Warning, Green=Healthy)
- Add **heatmap** on P95 column (darker = slower)

***

#### Panel 4C: Response Time Heatmap (Distribution by Time of Day)

**Purpose**: Spot if latency correlates with time/traffic patterns (e.g., slower at peak hours).[^5][^3]

**Visualization**: Heatmap (requires binning both time and duration)

**KQL Query**:

```kql
let lookback = 7d;
requests
| where timestamp > ago(lookback)
| extend HourOfDay = hourofday(timestamp),
         DurationBucket = case(
             duration < 500, "<500ms",
             duration < 1000, "500-1000ms",
             duration < 3000, "1-3s",
             duration < 5000, "3-5s",
             ">=5s"
         )
| summarize RequestCount = count() by HourOfDay, DurationBucket
| render columnchart kind=stacked
```

**Alternative (Grid Heatmap)**: Use Azure Workbook's built-in heatmap visualization; set `HourOfDay` as rows, `DurationBucket` as columns, and `RequestCount` as cell value.[^12]

***

### Section 5: TOP 10 SLOW CALLS (3 panels)

#### Panel 5A: Top 10 Slowest SQL Queries

**Purpose**: Find SQL queries causing backend latency.[^1][^2]

**Visualization**: Table

**KQL Query**:

```kql
let lookback = 24h;
dependencies
| where timestamp > ago(lookback)
| where type == "SQL"
| summarize 
    CallCount = count(),
    AvgDuration = round(avg(duration), 2),
    P95Duration = round(percentile(duration, 95), 2),
    MaxDuration = round(max(duration), 2),
    FailureRate = round(100.0 * countif(success == false) / count(), 2),
    SampleQuery = take_any(data)  // Shows one sample SQL statement
    by target  // SQL server/database
| order by P95Duration desc
| take 10
```

**Enhanced Version** (if you want per-query, not per-target):[^2]

```kql
dependencies
| where timestamp > ago(24h) and type == "SQL"
| summarize 
    CallCount = count(),
    P95Duration = round(percentile(duration, 95), 2),
    FailureRate = round(100.0 * countif(success == false) / count(), 2)
    by data  // Full SQL statement
| order by P95Duration desc
| take 10
```

**Warning**: `data` contains full SQL; if you have many unique queries (due to inline params), this will explode cardinality. Use parameterized queries in your app.[^13]

***

#### Panel 5B: Top 10 Slowest Snowflake Queries

**Purpose**: Identify slow Snowflake queries impacting user latency.[^14][^1]

**Visualization**: Table

**KQL Query**:

```kql
let lookback = 24h;
dependencies
| where timestamp > ago(lookback)
| where type == "Snowflake" or (type == "HTTP" and target contains "snowflake")  // Adjust based on how you log Snowflake
| extend QueryId = tostring(customDimensions["SnowflakeQueryId"]),
         WarehouseName = tostring(customDimensions["WarehouseName"])
| summarize 
    CallCount = count(),
    AvgDuration = round(avg(duration), 2),
    P95Duration = round(percentile(duration, 95), 2),
    MaxDuration = round(max(duration), 2),
    FailureRate = round(100.0 * countif(success == false) / count(), 2),
    SampleQueryId = take_any(QueryId)
    by name, WarehouseName
| order by P95Duration desc
| take 10
```

**Action**: Copy `SampleQueryId` → run in Snowflake:[^14]

```sql
SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY 
WHERE QUERY_ID = '<paste_query_id>'
```

to get full execution plan, bytes scanned, warehouse credits, etc.

***

#### Panel 5C: Top 10 Slowest Downstream API/HTTP Calls

**Purpose**: Find slow external dependencies (third-party APIs, microservices).[^1][^2]

**Visualization**: Table

**KQL Query**:

```kql
let lookback = 24h;
dependencies
| where timestamp > ago(lookback)
| where type in ("HTTP", "Ajax")  // HTTP dependencies
| summarize 
    CallCount = count(),
    AvgDuration = round(avg(duration), 2),
    P95Duration = round(percentile(duration, 95), 2),
    MaxDuration = round(max(duration), 2),
    FailureRate = round(100.0 * countif(success == false) / count(), 2)
    by target, name
| order by P95Duration desc
| take 10
```


***

## 4. BONUS PANELS (Optional but Highly Useful)

#### Panel 6A: Dependency Failure Correlation (Which Dependency Causes Request Failures?)

**Purpose**: When requests fail, which dependency is to blame?[^9]

**Visualization**: Table

**KQL Query**:

```kql
let lookback = 1h;
requests
| where timestamp > ago(lookback) and success == false
| join kind=inner (
    dependencies
    | where timestamp > ago(lookback) and success == false
    | project operation_Id, FailedDependency = target, DependencyType = type, DependencyResultCode = resultCode
) on operation_Id
| summarize 
    FailedRequestCount = dcount(operation_Id),
    SampleEndpoint = take_any(name)
    by FailedDependency, DependencyType, DependencyResultCode
| order by FailedRequestCount desc
```

**Output**: Shows "SQL DB timeout caused 23 failed requests to POST /api/orders"[^9]

***

#### Panel 6B: Slow Request + Slow Dependency Drill-Down

**Purpose**: For slow requests, see which dependency contributed most latency.[^10][^2]

**Visualization**: Table

**KQL Query**:

```kql
let lookback = 1h;
let slowThreshold = 5000;  // 5 seconds
requests
| where timestamp > ago(lookback) and duration > slowThreshold
| join kind=leftouter (
    dependencies
    | where timestamp > ago(lookback)
    | summarize 
        DependencyDuration = sum(duration),
        SlowestDependency = arg_max(duration, target)
        by operation_Id
) on operation_Id
| project 
    timestamp,
    Endpoint = name,
    TotalDuration = duration,
    DependencyDuration,
    OwnDuration = duration - DependencyDuration,  // Time spent in app code
    SlowestDependency = SlowestDependency_target,
    operation_Id
| order by TotalDuration desc
| take 20
```

**Interpretation**:[^10]

- If `OwnDuration` is high → app logic is slow (CPU/memory/GC)
- If `DependencyDuration` is high → external call is the bottleneck

***

#### Panel 6C: Availability SLO Tracker (Error Budget Burn Rate)

**Purpose**: Track how fast you're consuming error budget; alert if burn rate too high.[^15]

**Visualization**: Stat tiles + time series

**KQL Query**:

```kql
let lookback = 30d;
let sloTarget = 99.9;  // 99.9% availability
requests
| where timestamp > ago(lookback)
| summarize 
    TotalRequests = count(),
    FailedRequests = countif(success == false),
    AvailabilityPct = round(100.0 * countif(success == true) / count(), 3)
| extend 
    SLOTarget = sloTarget,
    SLOMet = AvailabilityPct >= sloTarget,
    ErrorBudget = TotalRequests * (100 - sloTarget) / 100,
    ErrorBudgetConsumed = FailedRequests,
    ErrorBudgetRemainingPct = round(100.0 * (ErrorBudget - FailedRequests) / ErrorBudget, 2)
```

**Stat Tiles**:[^16]

- Tile 1: **Availability %** (big number, green if SLO met, red if breached)
- Tile 2: **Error Budget Remaining %** (visual bar showing how much budget left)
- Tile 3: **Failed Requests This Month** (absolute count)

**Time Series** (burn rate over time):

```kql
requests
| where timestamp > ago(30d)
| summarize 
    FailureRate = 100.0 * countif(success == false) / count()
    by bin(timestamp, 1d)
| render timechart 
```


***

## 5. Validation (what to check, where, expected signals)

1. **Run each query in Logs first**: Validate data appears and row counts make sense.[^2]
2. **Check cardinality**: If any table has >100 rows, you may have cardinality explosion (e.g., SQL `data` with inline params).[^13]
3. **Verify custom dimensions**: For Snowflake/custom tracking, confirm `customDimensions["SnowflakeQueryId"]` exists; if not, adjust parsing.[^7][^8]
4. **Test time ranges**: Use workbook parameters `{TimeRange}` so users can switch from 1h to 7d without editing queries.[^5]

***

## 6. Rollback / Safety (only if changes are suggested)

No runtime changes; this is read-only telemetry analysis.[^17]

***

## 7. Next Steps (who does what)

### Immediate (today)

1. **App SRE**: Open Azure Portal → **Monitor → Workbooks → New**.[^17]
2. **App SRE**: Add **Section 1** (Endpoint Summary table from previous response) as first panel.
3. **App SRE**: Add **Section 2A-2D** (Failure Analysis panels) below it; validate data appears.

### Short-term (this week)

4. **App SRE**: Add **Section 3** (Throughput) and **Section 4** (Response Time).[^3][^2]
5. **App SRE**: Add **Section 5** (Top 10 Slow SQL/Snowflake/API).[^1]
6. **Backend team**: If Snowflake queries don't show up, confirm you're logging them with `TrackDependency` and adding `SnowflakeQueryId` to `customDimensions`.[^8][^1]

### Medium-term (2 weeks)

7. **App SRE**: Add **conditional formatting** to tables (red/yellow/green for health status).[^4][^11]
8. **App SRE**: Add **time range parameter** at the top of workbook so users can dynamically change lookback (1h/6h/24h/7d/30d).[^5]
9. **App SRE**: Add **deployment annotations** to time-series charts (mark when releases happen so you can correlate perf changes).[^3]
10. **App SRE**: Create **drill-through links**: Click an endpoint in the summary table → jump to a detailed page showing only that endpoint's latency/errors/dependencies.[^17]

### Long-term (1 month)

11. **App SRE**: Set up **scheduled exports** or **alerts** from workbook queries (e.g., "Alert me if any endpoint's p95 > 5s for 10 min").[^18]
12. **All teams**: Use the dashboard in your **weekly SRE review** to track SLO compliance, error budget burn, and capacity trends.[^15]

***

### Final Dashboard Summary (All 15 Panels)

| Section | Panel | Visualization | Purpose |
| :-- | :-- | :-- | :-- |
| **1. Summary** | Endpoint Table | Table | Total/Passed/Failed/4xx/5xx/Failure % per endpoint [^18] |
| **2. Failure** | 2A. Failure Trend | Line chart | Errors over time [^3] |
|  | 2B. Top 10 Failing | Bar chart | Worst endpoints by error count [^2] |
|  | 2C. Exception Breakdown | Table | Exception types \& counts [^7] |
|  | 2D. Failed Requests + Exceptions | Table | Correlated failures with root cause [^9] |
| **3. Throughput** | 3A. Request Rate Trend | Area chart | Requests per minute over time [^3] |
|  | 3B. Top 10 by Volume | Bar chart | Hottest endpoints [^2] |
| **4. Response Time** | 4A. Percentiles Over Time | Line chart | p50/p95/p99 latency [^2] |
|  | 4B. Latency by Endpoint | Table | Compare latency across endpoints [^2] |
|  | 4C. Response Time Heatmap | Heatmap | Latency distribution by hour [^3] |
| **5. Slow Calls** | 5A. Top 10 Slow SQL | Table | Slowest SQL queries [^1] |
|  | 5B. Top 10 Slow Snowflake | Table | Slowest Snowflake queries [^14] |
|  | 5C. Top 10 Slow HTTP | Table | Slowest downstream APIs [^2] |
| **6. Bonus** | 6A. Dependency Failure Correlation | Table | Which dependency caused failures [^9] |
|  | 6B. Slow Request Drill-Down | Table | App vs dependency latency breakdown [^10] |
|  | 6C. SLO/Error Budget Tracker | Stat tiles + chart | Availability \& burn rate [^15] |


***

**Questions to finalize**:

1. How is Snowflake logged in dependencies—`type="Snowflake"` or `type="HTTP"` with target containing "snowflake"?
2. Do you want a **single workbook with all 15 panels**, or separate workbooks per section (Failure, Throughput, Latency, Slow Calls)?
3. Should I add **drill-through actions** (click an endpoint → see detailed trace + logs)?

Let me know and I'll refine the exact queries/formatting!
<span style="display:none">[^19][^20][^21][^22][^23][^24][^25][^26][^27][^28]</span>

<div align="center">⁂</div>

[^1]: https://learn.microsoft.com/en-us/azure/azure-monitor/app/dependencies

[^2]: https://yurimelo.substack.com/p/mastering-kusto-query-language-kql

[^3]: https://docs.azure.cn/en-us/azure-monitor/visualize/workbooks-chart-visualizations

[^4]: https://iwconnect.com/efficient-log-management-visualizing-azure-logs-from-diverse-sources-using-monitor-and-log-analytics-workbooks/

[^5]: https://stackoverflow.com/questions/77759347/azure-monitor-workbook-using-bin

[^6]: https://azuretechinsider.com/advanced-kql-queries-logic-apps-application-insights/

[^7]: https://stackoverflow.com/questions/62075695/azure-application-insights-query-customdimensions

[^8]: https://stackoverflow.com/questions/72432774/azure-application-insights-kql-get-exception-details

[^9]: https://n8n.io/workflows/12803-track-azure-api-and-service-bus-failures-with-application-insights-correlation/

[^10]: https://stackoverflow.com/questions/65303372/how-to-evaluate-application-insights-requests-own-duration-without-considerin

[^11]: https://www.softensity.com/blog/understanding-azure-workbooks/

[^12]: https://learn.microsoft.com/en-us/azure/azure-monitor/visualize/workbooks-map-visualizations

[^13]: https://stackoverflow.com/questions/52905652/include-sql-query-parameter-values-in-application-insights-telemetry

[^14]: https://www.datadoghq.com/blog/snowflake-monitoring-tools/

[^15]: https://learn.microsoft.com/en-us/azure/well-architected/service-guides/application-insights

[^16]: https://docs.azure.cn/en-us/azure-monitor/visualize/workbooks-stat-visualizations

[^17]: https://learn.microsoft.com/en-us/azure/azure-monitor/visualize/workbooks-data-sources

[^18]: https://learn.microsoft.com/en-us/azure/azure-monitor/app/metrics-overview

[^19]: image.jpg

[^20]: https://github.com/Azure/data-api-builder/discussions/2096

[^21]: https://community.dynamics.com/blogs/post/?postid=1f46d24b-9a84-403b-88df-a98f4d845b34

[^22]: https://docs.azure.cn/en-us/azure-monitor/reference/queries-by-table

[^23]: https://github.com/MicrosoftDocs/azure-monitor-docs/blob/main/articles/azure-monitor/visualize/workbooks-chart-visualizations.md

[^24]: https://stackoverflow.com/questions/66504403/differences-between-insights-response-time-and-app-service-response-time

[^25]: https://learn.microsoft.com/en-us/azure/azure-monitor/app/data-model-complete

[^26]: https://www.bounteous.com/insights/2021/05/04/structured-logging-microsofts-azure-application-insights/

[^27]: https://learn.microsoft.com/en-us/azure/azure-functions/analyze-telemetry-data

[^28]: https://world.optimizely.com/blogs/K-Khan-/Dates/2025/12/troubleshooting-with-azure-application-insights-using-kql/
