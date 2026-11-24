# Limitations

SPL features with no direct DQL equivalent. For each one: what's missing, the closest DQL, and when to rethink the approach instead of forcing a translation.

## No equivalent, workaround exists

| SPL | What's missing in DQL | Closest DQL |
|---|---|---|
| `eventstats` | Adding an aggregate to every event without collapsing them | Compute the aggregate in a subquery and attach it with `lookup` ([example 9](examples.md#9-eventstats-compare-each-event-to-its-group)). Reads the data twice. |
| `transaction` | Session splitting by `maxspan`, `maxpause`, `startswith`, `endswith` | `summarize` by the session key with `min(timestamp)`, `max(timestamp)`, `count()`. One row per key for the whole range ([example 10](examples.md#10-transaction-session-summary-not-11)). |
| `streamstats` running total | Per-event running values | On time buckets: `makeTimeseries` + `arrayCumulativeSum()`. Per-event values have no equivalent. |
| `streamstats window=N` | Per-event moving windows | On time buckets: `arrayMovingAvg(x, N)`, `arrayMovingSum`, `arrayMovingMin`, `arrayMovingMax`. Window max is 60 buckets. |
| `delta` | Difference to the previous event | On time buckets: `arrayDelta()`. |
| `chart ... over a by b`, `xyseries`, `transpose` | Pivoting rows into columns | DQL returns long format (one row per a,b). Dashboard tiles can show it as a chart. |
| `appendcols` | Gluing results side by side by row position | `lookup` or `join` on a shared key. |
| `dedup N f` | Keeping N records per key | `dedup` keeps one. For a small N, `summarize` with `collectArray(...)` then `expand` with `limit:` can work. |
| `rex mode=sed` with backreferences | Regex replace with capture groups | `replaceString` / `replacePattern` without backreferences, or `parse` the parts and `concat` them. |
| Search-time field extraction (`props.conf`, `transforms.conf`) | Fields extracted automatically at query time | `parse` in every query, or extract once at ingest with OpenPipeline. |
| Automatic lookups | Lookups applied to every search | Add `lookup` to each query, or enrich at ingest. |
| KV store and external lookups | Lookup backends other than files | Upload the data as lookup data and `load` it, or enrich at ingest. |

## Not a query feature

These parts of Splunk don't belong in DQL. Tell the user what was dropped and where it belongs in Dynatrace.

| SPL | Where it goes in Dynatrace |
|---|---|
| Macros, eventtypes, tags | Expand them into the query before converting. |
| `outputlookup`, `collect`, summary indexing | DQL doesn't write data. Use OpenPipeline (for example, log metrics) or a Workflow. |
| `sendemail`, alert actions | Workflows and alerting. |
| `map`, `foreach`, `script`, custom search commands | No equivalent. Rewrite the logic, or run it in a Workflow. |
| `tstats` and accelerated data models | No acceleration layer. Use metrics for counts you need often. |
| Real-time searches | Use alerting on metrics or events instead of a constantly running query. |
| `rest`, `metadata`, `dbinspect` | Splunk internals. `fetch dt.system.buckets` lists Grail buckets. |

## How to say it in a conversion

When part of the query can't be converted, put it first in the answer, not at the end:

> This can't be converted 1:1. `transaction maxspan=30m` splits long sessions into several transactions, and DQL has no way to do that. The query below returns one row per session for the whole time range. Sessions longer than 30 minutes are where the results will differ.

Then give the closest query and say how to check the gap.
