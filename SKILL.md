---
name: splunk-to-dql
description: Convert Splunk SPL searches, alerts, and dashboard panels into Dynatrace Query Language (DQL) for Grail, explain where the results can differ, and recommend OpenPipeline log processing when fields should be extracted at ingest. Use when the user shares an SPL query, asks to migrate Splunk content or props.conf/transforms.conf to Dynatrace, wants the DQL equivalent of a Splunk command or function, or asks how to set up OpenPipeline for logs.
license: MIT
metadata:
  author: Mohsin Zaheer
  version: "0.1.0"
---

# Splunk SPL to Dynatrace DQL

Translate SPL into DQL that gives the same answer on Grail data, and say clearly where it can't.

Getting a query that runs is the easy part. SPL and DQL disagree on case sensitivity, nulls, operator precedence, default limits, and which record `dedup` keeps. A literal translation usually runs without errors and returns different numbers. Most of the work is catching those differences.

## Check for a Dynatrace connection

At the start, check whether you have a tool that runs DQL against the user's Dynatrace environment, such as a Dynatrace MCP server or the `dtctl` CLI. Tool names differ between setups. If you can't tell, ask.

- **Connected:** look things up instead of asking. Check field names, sample records, volume, and which pipeline handles the logs, then run the converted DQL and the `data record` checks. Follow [environment-discovery](references/environment-discovery.md): short timeframes, a `limit` on every query, read-only.
- **Not connected:** work from the SPL and what the user tells you. Use placeholders for unknown values, and give the user short queries to run when a fact changes the answer.

Either way, never copy the user's real data into files, issues, or examples.

## Before translating

1. **Expand knowledge objects.** If the SPL uses macros (`` `name` ``), eventtypes, tags, or saved searches, ask for the expanded search. In Splunk Web, Ctrl+Shift+E (Cmd+Shift+E on Mac) in the search bar shows it. Don't guess what a macro does.
2. **Get the data mapping.** Splunk `index`, `sourcetype`, `source`, and `host` have no fixed Grail equivalent. Depending on the environment they map to a bucket, `log.source`, `host.name`, `dt.entity.host`, `k8s.namespace.name`, or a custom attribute. Field names have the same problem: a field Splunk extracts at search time may not exist in Grail. If the user hasn't told you, write placeholders like `<payments log filter>` and list them under Assumptions. Never present an invented attribute name as real.
3. **Pick the right source.** Logs: `fetch logs`. Spans: `fetch spans`. Business events: `fetch bizevents`. Events: `fetch events`. Security events: `fetch security.events`. Davis problems: `fetch dt.davis.problems`. Metrics (`mstats`): `timeseries`. Lookup CSVs: `load "/lookups/<name>"`.

## Workflow

1. Split the SPL into stages: time range, base search, extractions, evals, filters, aggregations, joins, sort and limits, output fields. Note the intent (raw events, count by X, time series, top N, correlation).
2. Translate each stage using [command-mapping](references/command-mapping.md) and [function-mapping](references/function-mapping.md).
3. Go through every item in "Checks that change results" below.
4. Decide whether the query should stay a query or whether part of it belongs in OpenPipeline (next section).
5. Answer in the output format.

## Query or OpenPipeline

Query-time `parse` is fine for one-off searches. Recommend extracting at ingest with OpenPipeline when the SPL or the request shows any of these:

- `rex`, `spath`, `kv`, or `extract` in a dashboard panel, alert, or saved search that runs often
- the same extraction in several of the user's searches
- an alert or panel that only counts lines: suggest a counter metric
- masking with `rex mode=sed`: masking in a query doesn't change stored data
- most matched events being thrown away, or nullQueue drops in Splunk

If the SPL doesn't show how the search is used, ask: ad hoc, dashboard, or alert, and roughly how many logs per day. When connected, check volume yourself. When the user shares `props.conf` or `transforms.conf`, use [splunk-ingest-to-openpipeline](references/splunk-ingest-to-openpipeline.md). For setup steps, processors, and limits, see [openpipeline](references/openpipeline.md).

When you recommend OpenPipeline, give three parts:

1. **Use today:** the DQL with `parse`.
2. **Recommended setup:** the pipeline processors and route, with a sample record to test with **Run sample data**. DQL processors only allow `parse`, `fieldsAdd`, and other single-record commands. Move filters into the matching condition.
3. **After setup:** the shorter query that reads the extracted fields directly.

## Checks that change results

- **Precedence in the base search.** In the SPL `search` command, OR is evaluated before AND, so `index=app level=ERROR OR level=WARN` means `index=app AND (level=ERROR OR level=WARN)`. DQL evaluates `and` before `or`, like SPL `where` and `eval`. Always add parentheses.
- **Case.** SPL search terms and field values match case-insensitively. DQL `==`, `contains()`, and `startsWith()` are case-sensitive. For `field=value` use `matchesValue(field, "value")`, which is case-insensitive and supports `*`. For keywords use `matchesPhrase(content, "...")` or `content ~ "..."`. SPL `where` and `eval` comparisons are case-sensitive, so keep `==` for those.
- **Keywords.** A bare SPL keyword is a token search over `_raw`. Translate it to `matchesPhrase(content, "...")` or `content ~ "..."`, not `contains()`.
- **Missing group-by values.** SPL `stats ... by X` leaves out events where X is missing. DQL `summarize ... by: {X}` keeps them as a `null` group. Add `filter isNotNull(X)` first. `timechart` is the exception: it keeps a NULL series by default, as DQL does.
- **`!=` versus NOT.** SPL `field!=v` drops events that don't have the field, and `NOT field=v` keeps them. In DQL, `filter field != "v"` drops nulls and `filterOut field == "v"` keeps them.
- **Search-time extraction.** Splunk pulls `key=value` pairs out of raw events automatically. Grail doesn't. Unless the field was extracted at ingest (OpenPipeline) or is a known attribute, add a `parse` stage.
- **Types.** A string compared with a number gives null in DQL, and the record is filtered out. Cast with `toLong()` or `toDouble()`. Dividing a long by a long is integer division, so write `toDouble(a) / b`.
- **dedup.** SPL `dedup` keeps the first event in search order (newest first for historical searches) and drops events missing the field. DQL `dedup` keeps an arbitrary record unless you pass `sort:`. Write `filter isNotNull(x) | dedup x, sort: {timestamp desc}`.
- **Stage order.** Keep the SPL order of filter, dedup, and limit. Running dedup and then filter returns different rows than filter and then dedup.
- **Output order.** SPL `stats` output is sorted by the by-fields. DQL `summarize` output has no guaranteed order. If the SPL relies on that order (for example `stats ... | head 5`), add an explicit `sort`. Also add a tie-breaker to `sort` when ties would change which rows `limit` keeps.
- **Hidden SPL limits.** `sort` without a count returns at most 10,000 rows. `top`, `rare`, and `head` default to 10. `timechart ... by X` keeps the top 10 series and folds the rest into OTHER. `join` matches at most one right-side row per left row (`max=1`) and reads at most 50,000 subsearch rows. DQL applies none of these limits, so reproduce them on purpose or point them out.
- **Joins.** SPL `join` defaults to an inner join. DQL `join` returns every match, which can multiply rows. DQL `lookup` keeps every left row and takes the first match, which is the closest to SPL `join type=left`.
- **Statistics.** SPL `stdev` and `var` are sample statistics. DQL `stddev` and `variance` are population statistics. SPL `dc` is exact, but DQL `countDistinct` is an alias for `countDistinctApprox`. Use `countDistinctExact`, which fails above 1M distinct values. Percentiles are estimates in both languages.
- **Functions that look the same.** SPL `log(x)` is base 10, DQL `log(x)` is natural log, so use `log10`. SPL `substr` is 1-based and takes a length. DQL `substring` is 0-based and takes an exclusive end index. SPL `len` counts UTF-8 code points and DQL `stringLength` counts UTF-16 code units.
- **Regex.** DQL `parse` uses Dynatrace Pattern Language (DPL), which isn't regex. Rewrite `rex` patterns with DPL matchers such as `LD`, `INT`, `LONG`, `DOUBLE`, `IPV4`, `WORD`, and `JSON`. A DPL pattern matches from the start of the string, so a `rex` pattern that matches mid-line needs a leading `LD`, or `DATA` for multiline input. Fields the pattern doesn't match come back null.
- **Time.** `_time` becomes `timestamp`. `earliest=-24h` becomes `fetch logs, from: -24h`. `earliest=-1d@d` becomes `from: -1d@d`. With no explicit range the query uses the UI timeframe, so say that. Bucket edges in `timechart` and `makeTimeseries` can be aligned differently, so the first and last buckets may differ.
- **Charts.** When the SPL is a `timechart ... by X` shown as a column chart, ask whether the bars should stack or sit side by side. A Dynatrace Bar chart on `makeTimeseries` data always stacks. For side-by-side bars, use the query shape and tile settings in [visualization](references/visualization.md).

## Output format

1. **DQL** in a single `dql` code block, ready to paste into a Notebook. Put placeholders in `<angle brackets>`.
2. **What changed:** the mappings that aren't obvious, and every place where the result can differ from Splunk.
3. **Assumptions:** placeholders and field names the user must confirm.
4. **How to check it:** which numbers to compare against Splunk on the same fixed time window ([validation](references/validation.md)). When the logic has edge cases, include a short `data record(...)` query and its expected output, so the user can test the logic in the Dynatrace Playground without real data. When connected, run it and report the result.
5. **OpenPipeline (when it helps):** the three parts from "Query or OpenPipeline".

If part of the query can't be converted 1:1 (`transaction`, per-event `streamstats`, `foreach`, `map`, custom commands, wide `chart` tables), say so at the top, give the closest query, and explain the difference. See [limitations](references/limitations.md).

Keep the first version faithful to the SPL logic. Suggest other improvements, such as using a metric instead of counting logs, afterwards and separately.

## References

- [references/command-mapping.md](references/command-mapping.md): SPL commands to DQL
- [references/function-mapping.md](references/function-mapping.md): eval and stats functions
- [references/examples.md](references/examples.md): worked conversions, each with a Playground check
- [references/dql-patterns.md](references/dql-patterns.md): DQL syntax reminders and performance
- [references/validation.md](references/validation.md): how to prove the DQL matches the SPL
- [references/limitations.md](references/limitations.md): what has no direct equivalent
- [references/visualization.md](references/visualization.md): Dashboard chart setup, including side-by-side bars
- [references/openpipeline.md](references/openpipeline.md): OpenPipeline for logs: when to use it, setup, processors, limits
- [references/splunk-ingest-to-openpipeline.md](references/splunk-ingest-to-openpipeline.md): `props.conf` and `transforms.conf` settings mapped to OpenPipeline
- [references/openpipeline-examples.md](references/openpipeline-examples.md): processor examples to test with Run sample data
- [references/environment-discovery.md](references/environment-discovery.md): queries to run when connected to Dynatrace, and the safety rules
