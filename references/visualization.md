# Dashboard charts

How to make a Dynatrace Dashboard tile look like the Splunk panel it replaces. This covers the one case that needs a different query: bars side by side.

## Side-by-side bars per time bucket

A Splunk `timechart ... by X` shown as a column chart puts the series side by side in each time bucket. In Dynatrace, a **Bar chart** tile on `makeTimeseries` or `timeseries` data always stacks series that share a unit, and the tile has no setting to change that.

The **Categorical bar chart** can group bars. It needs a different query shape: one row per time bucket, the bucket as a text label, and one number column per series. No timestamp column can be left in the result, otherwise the tile stays a time chart.

**SPL**
```spl
index=app sourcetype=app_logs (level=ERROR OR level=WARN)
| timechart span=5m count by level
```

**DQL**
```dql
fetch logs
| filter <app log filter>
| summarize {
    error = countIf(matchesValue(loglevel, "ERROR")),
    warn  = countIf(matchesValue(loglevel, "WARN"))
  }, by: {bucket = bin(timestamp, 5m)}
| sort bucket asc
| fieldsAdd time = formatTimestamp(bucket, format: "MM-dd HH:mm")
| fieldsRemove bucket
```

**Tile settings**
1. Visualization: **Categorical bar chart**. Not "Bar chart", which is the time series version.
2. Data mapping: Categories = `time`, Values = `error` and `warn`.
3. Turn on **Vertical layout**. Without it the bars are horizontal.
4. Set **Group mode** to grouped. The default is stacked.

**Notes**
- Sort by the timestamp before turning it into text. Text labels sort alphabetically, so sorting after the conversion can put buckets out of order.
- Pick a label format that matches the bucket size: `"MM-dd HH:mm"` for minutes or hours, `"yyyy-MM-dd"` for days.
- You need one `countIf` column per series, so you must know the series values in advance. Splunk creates one column per value on its own. When the values aren't known, run `summarize count(), by: {loglevel}` first to list them.
- `summarize` only returns buckets that contain events. Splunk `timechart` shows empty buckets as 0. Here an empty bucket is missing from the chart, and the axis doesn't show the gap.
- Removing the timestamp also removes time zoom on the tile, because the chart no longer has a time axis.

**Check**

Paste this into a new Dashboard tile and apply the tile settings above:

```dql
data record(time = "01-10 10:00", error = 5, warn = 3),
     record(time = "01-10 10:05", error = 2, warn = 6),
     record(time = "01-10 10:10", error = 4, warn = 4)
```

Expected: three groups of two vertical bars, side by side.
