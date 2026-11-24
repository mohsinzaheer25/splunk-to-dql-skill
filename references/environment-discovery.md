# Looking up the environment

Use this when you have a tool that runs DQL against the user's Dynatrace environment, such as a Dynatrace MCP server or the `dtctl` CLI. The point is to check facts instead of asking the user for them.

## Rules

- **Read only.** Run queries. Don't create, change, or delete settings, pipelines, buckets, or documents, even if a tool allows it. Give the user the steps and let them make the change.
- **Keep queries cheap.** Grail log queries are billed by data scanned. Every log query gets a short timeframe (start with `from: -15m` or `-1h`) and a `limit`. Prefer metrics and `fieldsSummary` over scanning `content`. Ask before running anything over more than 24 hours of logs or without a selective filter.
- **Say what you ran.** Show each query and summarize what it found, so the user can follow and repeat it.
- **Keep real data out of the repo.** Don't copy real field values, hostnames, IPs, or log lines into files, examples, or issues. Use them only to answer the user's question.
- **Ask when results are unclear.** If a lookup returns nothing or something unexpected, say so and ask. Don't guess.

## What to check, and how

Replace `<filter>` with the narrowest condition you have, for example `matchesValue(log.source, "*payments*")`.

### Which fields exist

```dql
fetch logs, from: -15m
| filter <filter>
| limit 5
```

Look at the field names and types, and at whether the values you need are already separate fields or only inside `content`. This one query answers most "what is the field called" questions.

To see how often fields are filled across more records:

```dql
fetch logs, from: -15m
| filter <filter>
| limit 1000
| fieldsSummary loglevel, log.source, host.name
```

### Where the Splunk index or sourcetype went

```dql
fetch logs, from: -15m
| summarize count = count(), by: {log.source, dt.system.bucket}
| sort count desc
| limit 30
```

Match the user's Splunk index or sourcetype to a `log.source`, bucket, or other attribute. Confirm the match with the user before relying on it.

### How much data arrives

Use the OpenPipeline self-monitoring metric. It's a metric, so it doesn't scan logs:

```dql
timeseries records = sum(dt.sfm.openpipeline.ingest_sources_in.records),
  by: {dt.openpipeline.source}, filter: {configuration == "logs"}, from: -24h
| fieldsAdd per_day = arraySum(records)
| fields dt.openpipeline.source, per_day
| sort per_day desc
```

This gives the volume per ingest source, not for one application. For a single filter, count over a short window and scale it up:

```dql
fetch logs, from: -1h
| filter <filter>
| summarize count = count()
```

Multiply by 24 for a rough daily count. Logs often vary through the day, so call it an estimate.

### Which pipeline handles the logs

```dql
fetch logs, from: -15m
| filter <filter>
| summarize count = count(), by: {dt.openpipeline.pipelines, dt.openpipeline.source}
```

- `logs:default` means the logs reach no custom pipeline, which is often the classic pipeline. See the classic pipeline notes in [openpipeline](openpipeline.md).
- A custom pipeline name means processing already exists. Ask who owns it before suggesting changes.

### Whether a field is already extracted

If the sample shows the field as its own column, use it directly and skip `parse`. If it's only inside `content`, the query needs `parse` today. Consider OpenPipeline if the query runs often.

### Whether records are being dropped

```dql
timeseries records = sum(dt.sfm.openpipeline.not_stored.records),
  by: {pipeline_id, reason}, from: -24h
```

Use this when the user reports missing logs after a pipeline change.

## Checking a conversion

1. **Run the `data record` check** from the answer. It costs nothing because it reads no stored data. Compare the result with the expected table.
2. **Run the converted query** on a short window with a `limit`. Check that it returns rows, that the fields have values, and that the types are right (numbers aren't strings).
3. **Tell the user what to compare in Splunk.** You can only see the Dynatrace side. Give them the SPL and the window, and list the numbers to compare, as in [validation](validation.md).
4. **Report** what ran, what came back, and anything that looks wrong.

## When there is no connection

Give the user the same queries from this file and ask them to run them and share the result, with real values replaced if they plan to post it publicly. Keep it to the one or two queries that change the answer.
