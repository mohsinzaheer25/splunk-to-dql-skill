# Worked examples

Each example has four parts:

- **SPL:** the Splunk search.
- **DQL:** the translation to run on real data. `<angle brackets>` mark values you must fill in for your environment, such as the filter that selects the right logs.
- **Notes:** why the DQL is written this way.
- **Check:** a self-contained query built on `data record(...)`. It runs the same logic on a few made-up rows, so you can paste it into a Notebook without any log data. If you don't have a Dynatrace environment, open any example on the [DQL docs](https://docs.dynatrace.com/docs/platform/grail/dynatrace-query-language/commands), click **Run in Playground**, and replace the query. The expected result is listed under it.

All hosts, IPs, and IDs below are made up. The IPs come from the documentation ranges `192.0.2.0/24` and `198.51.100.0/24`.

---

## 1. Count errors by status

**SPL**
```spl
index=web sourcetype=access_combined status>=400
| stats count by status
| sort -count
```

**DQL**
```dql
fetch logs
| filter <web access log filter>
| filter status >= 400
| summarize count = count(), by: {status}
| sort count desc
```

**Notes**
- This assumes `status` is a numeric field in Grail. If it only exists inside `content`, extract it first with `parse content, "<DPL pattern>"`. If it's a string, compare `toLong(status) >= 400`.
- `status >= 400` is null when `status` is missing, so those records are dropped. That gives the same result as SPL dropping events without `status` from `stats ... by status`, so no `isNotNull` is needed here.

**Check**
```dql
data record(status = 200), record(status = 404), record(status = 404),
     record(status = 500), record()
| filter status >= 400
| summarize count = count(), by: {status}
| sort count desc
```

| status | count |
|---|---|
| 404 | 2 |
| 500 | 1 |

---

## 2. OR in the base search

**SPL**
```spl
index=app host=web-01 level=error OR level=warn
| table _time host level
```

**DQL**
```dql
fetch logs
| filter <app log filter>
| filter host.name == "web-01"
    and (matchesValue(loglevel, "error") or matchesValue(loglevel, "warn"))
| fields timestamp, host.name, loglevel
```

**Notes**
- The SPL `search` command evaluates OR before AND, so this SPL means `host=web-01 AND (level=error OR level=warn)`. DQL evaluates `and` first, so the parentheses are required.
- In the base search, SPL matches field values without regard to case, so `level=error` also matches `ERROR`. `matchesValue()` matches the same way. `loglevel == "error"` would not.
- The field names `host.name` and `loglevel` are assumptions. Use whatever fields hold the host and level in your environment.

**Check**
```dql
data record(host.name = "web-01", loglevel = "ERROR"),
     record(host.name = "web-01", loglevel = "warn"),
     record(host.name = "web-01", loglevel = "INFO"),
     record(host.name = "web-02", loglevel = "WARN")
| filter host.name == "web-01"
    and (matchesValue(loglevel, "error") or matchesValue(loglevel, "warn"))
```

| host.name | loglevel |
|---|---|
| web-01 | ERROR |
| web-01 | warn |

Without the parentheses, the `web-02` / `WARN` row comes back too.

---

## 3. rex extraction, top sources

**SPL**
```spl
index=security sourcetype=firewall action=blocked
| rex field=_raw "src=(?<src_ip>\d+\.\d+\.\d+\.\d+)"
| stats count by src_ip
| sort -count
| head 20
```

**DQL**
```dql
fetch logs
| filter <firewall log filter>
| parse content, "LD 'action=' WORD:action"
| filter matchesValue(action, "blocked")
| parse content, "LD 'src=' IPV4:src_ip"
| filter isNotNull(src_ip)
| summarize count = count(), by: {src_ip}
| sort count desc, src_ip asc
| limit 20
```

**Notes**
- Splunk extracts `action=blocked` on its own at search time. Grail doesn't, so `action` needs its own `parse` (unless it was extracted at ingest).
- A DPL pattern matches from the start of the string. The leading `LD` skips text until the literal `'src='`, which does what an unanchored regex does. `LD` needs at least one character before the literal, so if the key can be the very first thing in the line, use `LD{0,4096}` instead.
- `filter isNotNull(src_ip)` reproduces SPL `stats` dropping events where the extraction failed.
- `src_ip asc` is a tie-breaker. Without it, which equal-count rows survive `limit 20` can change between runs.

**Check**
```dql
data record(content = "2026-01-10T10:00:00Z action=blocked src=192.0.2.10 dst=198.51.100.1"),
     record(content = "2026-01-10T10:01:00Z action=BLOCKED src=192.0.2.10 dst=198.51.100.1"),
     record(content = "2026-01-10T10:02:00Z action=allowed src=192.0.2.11 dst=198.51.100.1"),
     record(content = "2026-01-10T10:03:00Z action=blocked src=192.0.2.12 dst=198.51.100.1"),
     record(content = "2026-01-10T10:04:00Z action=blocked dst=198.51.100.1")
| parse content, "LD 'action=' WORD:action"
| filter matchesValue(action, "blocked")
| parse content, "LD 'src=' IPV4:src_ip"
| filter isNotNull(src_ip)
| summarize count = count(), by: {src_ip}
| sort count desc, src_ip asc
```

| src_ip | count |
|---|---|
| 192.0.2.10 | 2 |
| 192.0.2.12 | 1 |

---

## 4. Error rate per host

**SPL**
```spl
index=web sourcetype=access_combined
| eval is_error = if(status >= 500, 1, 0)
| stats sum(is_error) as errors, count as total by host
| eval error_rate = round(errors / total * 100, 2)
| where error_rate > 5
| sort -error_rate
```

**DQL**
```dql
fetch logs
| filter <web access log filter>
| filter isNotNull(host.name)
| summarize {errors = countIf(status >= 500), total = count()}, by: {host.name}
| fieldsAdd error_rate = round(toDouble(errors) / total * 100, decimals: 2)
| filter error_rate > 5
| sort error_rate desc
```

**Notes**
- `countIf` replaces the `eval` + `sum` pair.
- `toDouble()` is required. `errors` and `total` are both longs, and long divided by long is integer division in DQL, so `1 / 4` gives `0`.

**Check**
```dql
data record(host.name = "web-01", status = 200), record(host.name = "web-01", status = 500),
     record(host.name = "web-02", status = 200), record(host.name = "web-02", status = 200),
     record(host.name = "web-02", status = 200), record(host.name = "web-02", status = 503),
     record(host.name = "web-03", status = 200)
| filter isNotNull(host.name)
| summarize {errors = countIf(status >= 500), total = count()}, by: {host.name}
| fieldsAdd error_rate = round(toDouble(errors) / total * 100, decimals: 2)
| filter error_rate > 5
| sort error_rate desc
```

| host.name | errors | total | error_rate |
|---|---|---|---|
| web-01 | 1 | 2 | 50 |
| web-02 | 1 | 4 | 25 |

Remove `toDouble()` and every rate becomes 0, so no rows pass the filter.

---

## 5. Latest state per ID, then filter

**SPL**
```spl
index=vuln sourcetype=scanner
| dedup vuln_id
| where status="OPEN"
| table _time vuln_id host status
```

**DQL**
```dql
fetch logs
| filter <scanner log filter>
| filter isNotNull(vuln_id)
| dedup vuln_id, sort: {timestamp desc}
| filter status == "OPEN"
| fields timestamp, vuln_id, host.name, status
```

**Notes**
- SPL `dedup` keeps the newest event per ID and drops events without `vuln_id`. DQL `dedup` keeps a random record unless you pass `sort:`, and keeps nulls.
- Keep the order: dedup first, then filter. Filtering first returns old `OPEN` records for IDs that were fixed later.
- `where` is case-sensitive in SPL, so `==` is the correct translation here.

**Check**
```dql
data record(timestamp = toTimestamp("2026-01-10T10:00:00Z"), vuln_id = "V-1", status = "OPEN"),
     record(timestamp = toTimestamp("2026-01-11T10:00:00Z"), vuln_id = "V-1", status = "FIXED"),
     record(timestamp = toTimestamp("2026-01-10T10:00:00Z"), vuln_id = "V-2", status = "OPEN"),
     record(timestamp = toTimestamp("2026-01-09T10:00:00Z"), vuln_id = "V-2", status = "OPEN")
| filter isNotNull(vuln_id)
| dedup vuln_id, sort: {timestamp desc}
| filter status == "OPEN"
```

| timestamp | vuln_id | status |
|---|---|---|
| 2026-01-10T10:00:00Z | V-2 | OPEN |

With the filter before the dedup, `V-1` shows up too.

---

## 6. top, with the percent column

**SPL**
```spl
index=app sourcetype=app_logs level=ERROR
| top limit=3 service
```

**DQL**
```dql
fetch logs
| filter <app log filter> and matchesValue(loglevel, "ERROR")
| filter isNotNull(service)
| summarize events = count(), by: {service}
| summarize {total = sum(events), rows = collectArray(record(service = service, events = events))}
| expand rows
| fieldsAdd service = rows[service], count = rows[events],
    percent = round(toDouble(rows[events]) / total * 100, decimals: 2)
| fields service, count, percent
| sort count desc, service asc
| limit 3
```

**Notes**
- `top` returns `count` and `percent`. The second `summarize` computes the total across all services, so each row can get its percentage.
- If you don't need `percent`, stop after `summarize events = count(), by: {service} | sort events desc | limit 3`.
- Splunk shows more decimals in `percent`. Ties may be ordered differently than in Splunk.

**Check**
```dql
data record(service = "cart"), record(service = "cart"), record(service = "cart"),
     record(service = "auth"), record(service = "auth"),
     record(service = "search"), record(service = "pay"), record()
| filter isNotNull(service)
| summarize events = count(), by: {service}
| summarize {total = sum(events), rows = collectArray(record(service = service, events = events))}
| expand rows
| fieldsAdd service = rows[service], count = rows[events],
    percent = round(toDouble(rows[events]) / total * 100, decimals: 2)
| fields service, count, percent
| sort count desc, service asc
| limit 3
```

| service | count | percent |
|---|---|---|
| cart | 3 | 42.86 |
| auth | 2 | 28.57 |
| pay | 1 | 14.29 |

---

## 7. timechart with empty buckets

**SPL**
```spl
index=app sourcetype=app_logs level=ERROR
| timechart span=1m count
```

**DQL**
```dql
fetch logs
| filter <app log filter> and matchesValue(loglevel, "ERROR")
| makeTimeseries count = count(default: 0), interval: 1m
```

**Notes**
- Splunk shows 0 for minutes with no events. `makeTimeseries` fills them with null unless you set `default: 0`.
- With `by` fields, Splunk keeps only the top 10 series and folds the rest into OTHER. DQL keeps every series.

**Check**
```dql
data record(timestamp = toTimestamp("2026-01-10T10:00:10Z")),
     record(timestamp = toTimestamp("2026-01-10T10:00:40Z")),
     record(timestamp = toTimestamp("2026-01-10T10:02:05Z"))
| makeTimeseries count = count(default: 0), interval: 1m,
    from: toTimestamp("2026-01-10T10:00:00Z"), to: toTimestamp("2026-01-10T10:03:00Z")
```

Expected: `count` is `[2, 0, 1]`. Without `default: 0` it is `[2, null, 1]`.

---

## 8. Left join to a lookup table

**SPL**
```spl
index=app sourcetype=orders
| stats count as orders by customer_id
| join type=left customer_id [| inputlookup customers.csv | fields customer_id tier]
```

**DQL**
```dql
fetch logs
| filter <orders log filter>
| filter isNotNull(customer_id)
| summarize orders = count(), by: {customer_id}
| lookup [ load "/lookups/customers" ],
    sourceField: customer_id, lookupField: customer_id, fields: {tier}
```

**Notes**
- `customers.csv` must first be uploaded to Grail as lookup data. The path `/lookups/customers` is an example.
- SPL `join` takes one match per row (`max=1`). DQL `lookup` also takes the first match and keeps unmatched rows. DQL `join ..., kind: leftOuter` returns every match, so a duplicated key in the lookup table would double the row.
- Not yet confirmed: the docs don't say whether `fields: {tier}` keeps the default `lookup.` prefix. If the column comes back as `lookup.tier`, add `| fieldsRename tier = lookup.tier`.

**Check**
```dql
data record(customer_id = "c1"), record(customer_id = "c1"),
     record(customer_id = "c2"), record(customer_id = "c3")
| summarize orders = count(), by: {customer_id}
| lookup [ data record(customer_id = "c1", tier = "gold"),
                record(customer_id = "c2", tier = "silver"),
                record(customer_id = "c2", tier = "bronze") ],
    sourceField: customer_id, lookupField: customer_id, fields: {tier}
| sort customer_id asc
```

| customer_id | orders | tier |
|---|---|---|
| c1 | 2 | gold |
| c2 | 1 | silver |
| c3 | 1 | null |

---

## 9. eventstats: compare each event to its group

**SPL**
```spl
index=web sourcetype=access_combined
| eventstats avg(response_ms) as avg_ms by host
| where response_ms > 2 * avg_ms
| table _time host uri response_ms avg_ms
```

**DQL**
```dql
fetch logs, from: <start>, to: <end>
| filter <web access log filter>
| lookup [
    fetch logs, from: <start>, to: <end>
    | filter <web access log filter>
    | summarize avg_ms = avg(response_ms), by: {host.name}
  ], sourceField: host.name, lookupField: host.name, fields: {avg_ms}
| filter response_ms > 2 * avg_ms
| fields timestamp, host.name, uri, response_ms, avg_ms
```

**Notes**
- DQL has no `eventstats`. The subquery computes the per-host average, and `lookup` attaches it to every event.
- The data is read twice, which doubles the scan cost. Give both queries the same `from`/`to`, otherwise the averages come from a different window.
- Events without a host get no average and are dropped by the filter, as in SPL.
- Same prefix caveat as example 8: if the column comes back as `lookup.avg_ms`, rename it.

**Check**
```dql
data record(host.name = "web-01", uri = "/a", response_ms = 100),
     record(host.name = "web-01", uri = "/b", response_ms = 100),
     record(host.name = "web-01", uri = "/c", response_ms = 700),
     record(host.name = "web-02", uri = "/a", response_ms = 50)
| lookup [
    data record(host.name = "web-01", response_ms = 100),
         record(host.name = "web-01", response_ms = 100),
         record(host.name = "web-01", response_ms = 700),
         record(host.name = "web-02", response_ms = 50)
    | summarize avg_ms = avg(response_ms), by: {host.name}
  ], sourceField: host.name, lookupField: host.name, fields: {avg_ms}
| filter response_ms > 2 * avg_ms
```

| host.name | uri | response_ms | avg_ms |
|---|---|---|---|
| web-01 | /c | 700 | 300 |

---

## 10. transaction: session summary (not 1:1)

**SPL**
```spl
index=web sourcetype=access_combined
| transaction session_id maxspan=30m
| table session_id duration eventcount
```

**DQL**
```dql
fetch logs
| filter <web access log filter>
| filter isNotNull(session_id)
| summarize {first_seen = min(timestamp), last_seen = max(timestamp), eventcount = count()},
    by: {session_id}
| fieldsAdd duration = (last_seen - first_seen) / 1s
| fields session_id, duration, eventcount
```

**Notes**
- This is not 1:1. `maxspan=30m` splits a session into several transactions when it runs longer than 30 minutes. The DQL returns one row per `session_id` for the whole time range.
- Rows with `duration > 1800` are the ones where Splunk returns more than one transaction. Check those rows when you compare results.
- `/ 1s` turns the duration into seconds, matching Splunk's `duration` field.

**Check**
```dql
data record(session_id = "s1", timestamp = toTimestamp("2026-01-10T10:00:00Z")),
     record(session_id = "s1", timestamp = toTimestamp("2026-01-10T10:05:00Z")),
     record(session_id = "s1", timestamp = toTimestamp("2026-01-10T10:50:00Z")),
     record(session_id = "s2", timestamp = toTimestamp("2026-01-10T10:00:00Z"))
| summarize {first_seen = min(timestamp), last_seen = max(timestamp), eventcount = count()},
    by: {session_id}
| fieldsAdd duration = (last_seen - first_seen) / 1s
| fields session_id, duration, eventcount
| sort session_id asc
```

| session_id | duration | eventcount |
|---|---|---|
| s1 | 3000 | 3 |
| s2 | 0 | 1 |

Splunk would return three rows: `s1` split into a 2-event transaction (300 s) and a 1-event transaction, plus `s2`.

---

## 11. mstats to timeseries (template, no check)

**SPL**
```spl
| mstats avg(cpu.usage) as avg_cpu WHERE index=metrics span=5m BY host
```

**DQL**
```dql
timeseries avg_cpu = avg(dt.host.cpu.usage), by: {dt.entity.host}, interval: 5m
| fieldsAdd host = entityName(dt.entity.host)
```

**Notes**
- Metric keys and dimensions differ between Splunk and Dynatrace. `dt.host.cpu.usage` is the built-in Dynatrace host CPU metric. A custom Splunk metric needs its own mapping.
- This example needs real metric data, so it has no `data record` check.
