# DQL syntax reminders and performance

Short notes on DQL syntax that people coming from SPL often get wrong, plus the performance advice from the Dynatrace docs.

## Syntax

```dql
// Comments start with //
fetch logs, from: -2h                      // relative time
| filter loglevel == "ERROR"               // == for equality, "double quotes" for strings
| filter matchesValue(host.name, "web-*")  // case-insensitive value match with wildcard
| filter content ~ "timeout"               // case-insensitive token search
| parse content, """LD 'user=' WORD:user"""   // triple quotes avoid escaping
| fieldsAdd is_slow = duration > 1s        // duration literals: ms, s, m, h, d
| summarize {
    total = count(),
    slow  = countIf(is_slow)
  }, by: {host.name, user}
| fieldsAdd slow_pct = toDouble(slow) / total * 100
| sort slow_pct desc
| limit 20
```

- One aggregation: `summarize c = count(), by: {a}`. Several: wrap them in `{ }`.
- Named parameters use a colon: `by:`, `interval:`, `sort:`, `else:`, `decimals:`, `kind:`.
- Field names with special characters need backticks: `` `my-field` ``.
- Record and array access: `r[key]`, `r[key][sub]`, `a[0]`, `a[-1]`.
- Fields added in a `fieldsAdd` can't be used in the same `fieldsAdd`. Add another stage.
- `==` and `!=` return null if either side is null, and `filter` drops null. Use `isNull()`, `isNotNull()`, or `coalesce()` when nulls matter.
- `filterOut x` keeps records where `x` is null. `filter not x` drops them.

## Performance

Grail reads data at query time. Query cost depends on how much data is scanned, and on DPS licensing, log queries are billed per GiB scanned. These come from the [DQL best practices](https://docs.dynatrace.com/docs/platform/grail/dynatrace-query-language/dql-best-practices) page:

- Keep the time range as short as the question allows. Set `from:` in dashboards and alerts that should not follow the UI timeframe.
- Limit the buckets when you know them: `fetch logs, bucket: {"<bucket_name>"}`.
- Filter first. Put the most selective filter right after `fetch`.
- Filter on the field directly. `filter k8s.namespace.name ~ "astro*"` is faster than `filter matchesValue(lower(k8s.namespace.name), "astro*")`.
- Prefer positive filters over negations where possible.
- Don't use `join` or `lookup` just to filter. Filter on a field that was added at ingest.
- Drop fields you don't need early with `fields`, `fieldsKeep`, or `fieldsRemove`, especially `content` before a large `summarize`.
- `samplingRatio` and `scanLimitGBytes` make exploratory queries cheaper. Don't use them in queries you compare with Splunk or use for alerts, because the numbers will be partial.

## Make queries cheaper after migration

Once the DQL gives the same answer as Splunk, you can often reduce cost:

- Extract fields at ingest with OpenPipeline, so queries don't need `parse` on `content`.
- When an alert only counts log lines, create a log metric and query it with `timeseries`.
- Replace a lookup that never changes with an enrichment at ingest.

Do these as a second step, after the first version has been compared against Splunk.
