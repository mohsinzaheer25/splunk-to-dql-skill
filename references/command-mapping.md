# SPL commands to DQL

Each row comes from the Dynatrace DQL reference and the Splunk SPL Search Reference (links at the bottom). "Watch out" lists the cases where a direct translation returns different results.

## Getting data

| SPL | DQL | Watch out |
|---|---|---|
| `index=x sourcetype=y` | `fetch logs` + `filter` on whatever attribute holds that source in your environment (`log.source`, `k8s.namespace.name`, a bucket, a custom attribute) | No fixed mapping exists. Ask, or use a placeholder. |
| `earliest=-24h latest=now` | `fetch logs, from: -24h` | Without `from:`/`to:`, the UI timeframe applies. |
| `earliest=-1d@d latest=@d` | `fetch logs, from: -1d@d, to: @d` | `@` aligns to the unit, like Splunk snap-to. |
| fixed window | `fetch logs, timeframe: "2026-01-10T00:00:00Z/2026-01-11T00:00:00Z"` | Use this when comparing results with Splunk. |
| `\| mstats avg(m) WHERE ... span=5m BY host` | `timeseries avg(<metric.key>), by: {dt.entity.host}, interval: 5m` | Metric keys and dimensions differ between products. |
| `\| tstats count WHERE index=x BY host` | `fetch logs \| filter ... \| summarize count(), by: {...}` | Grail has no accelerated data models. Scan cost depends on the time range and filters. |
| `\| inputlookup file.csv` | `load "/lookups/<name>"` | The file must first be uploaded to Grail as lookup data. |
| `\| makeresults` / `format=csv data=...` | `data record(a = 1), record(a = 2)` or `data json: "..."` | `makeresults count=N` has no counter. Write N records. |
| `\| rest`, `\| metadata`, `\| eventcount` | No equivalent | `fetch dt.system.buckets` lists buckets. |

## Filtering

| SPL | DQL | Watch out |
|---|---|---|
| bare keyword `error` | `filter matchesPhrase(content, "error")` or `filter content ~ "error"` | Both match tokens case-insensitively, like SPL. `contains()` is case-sensitive and matches substrings. |
| `"connection refused"` | `filter matchesPhrase(content, "connection refused")` | |
| keyword across all fields | `search "error"` | The DQL `search` command searches every field. |
| `field=value` (in search) | `filter matchesValue(field, "value")` | Case-insensitive, supports `*`. Use `==` only if you know the case matches. |
| `field=val*` | `filter matchesValue(field, "val*")` | |
| `field IN (a, b)` | `filter in(field, {"a", "b"})` | `in()` is case-sensitive. |
| `field!=v` | `filter field != "v"` | Drops records where `field` is missing, as SPL does. |
| `NOT field=v` | `filterOut field == "v"` | Keeps records where `field` is missing, as SPL does. `filter not (...)` would drop them. |
| `A B OR C` (search) | `filter A and (B or C)` | SPL search evaluates OR before AND. DQL evaluates AND first. |
| `\| where x > 5` | `filter x > 5` | `where` is case-sensitive in both. Cast strings with `toLong`/`toDouble` first. |
| `\| where like(f, "a%")` | `filter like(f, "a%")` | Same `%` and `_` wildcards. |
| `\| where match(f, "regex")` | `filter matchesPattern(f, "<DPL>")` | DPL, not regex. Test the pattern against sample rows. |
| `\| regex f="..."` | `filter matchesPattern(f, "<DPL>")` | Same. |
| `\| where cidrmatch("10.0.0.0/8", ip)` | `filter ipIn(ip, "10.0.0.0/8")` | |
| `\| dedup f` | `filter isNotNull(f) \| dedup f, sort: {timestamp desc}` | SPL keeps the newest event and drops events missing `f`. DQL keeps a random record if you leave out `sort:`. |
| `\| dedup f keepempty=true` | `dedup f, sort: {timestamp desc}` | |
| `\| dedup f sortby -x` | `dedup f, sort: {x desc}` | |
| `\| dedup 3 f` | No direct equivalent | DQL `dedup` keeps one record per key. |

## Extracting fields

| SPL | DQL | Watch out |
|---|---|---|
| automatic `key=value` extraction | `parse content, "LD 'key=' NSPACE:key"` or extract at ingest | Grail has no search-time auto extraction. |
| `\| rex "user=(?<user>\w+)"` | `parse content, "LD 'user=' WORD:user"` | Rewrite each regex in DPL. All exported fields are null when the pattern doesn't match. |
| `\| rex field=f "..."` | `parse f, "..."` | |
| `\| rex mode=sed "s/a/b/g"` | `fieldsAdd f = replaceString(f, "a", "b")` or `replacePattern(f, "<DPL>", "b")` | No regex backreferences. |
| `\| spath` | `parse content, "JSON:j"`, then `j[key][sub]` | Or `fieldsAdd x = jsonField(content, "key")`. |
| `\| spath path=a.b output=x` | `parse content, "JSON:j" \| fieldsAdd x = j[a][b]` | |
| `\| extract` / `\| kv` | `parse` with `KVP{...}` matcher | |
| `\| makemv delim="," f` | `fieldsAdd f = splitString(f, ",")` | |
| `\| mvexpand f` | `expand f` | Empty arrays produce no records. |

## Changing fields

| SPL | DQL | Watch out |
|---|---|---|
| `\| eval a = x, b = y` | `fieldsAdd a = x, b = y` | When `b` uses `a`, put them in separate `fieldsAdd` stages. |
| `\| rename old AS new` | `fieldsRename new = old` | New name first. List each field; no wildcards. |
| `\| table a b c` | `fields a, b, c` | `fields` creates missing fields as null, like `table` shows empty columns. |
| `\| fields a b` | `fieldsKeep a, b` | |
| `\| fields - a b` | `fieldsRemove a, b` | |
| `\| fillnull value=0 a b` | `fieldsAdd a = coalesce(a, 0), b = coalesce(b, 0)` | SPL `fillnull` with no field list fills every field. In DQL you must name each one. |
| `\| convert num(x)` | `fieldsAdd x = toDouble(x)` | |

## Aggregating

| SPL | DQL | Watch out |
|---|---|---|
| `\| stats count by a` | `filter isNotNull(a) \| summarize count = count(), by: {a}` | SPL drops events missing `a`. DQL keeps them as a null group. SPL output is sorted by `a`. |
| `\| stats count` | `summarize count = count()` | |
| `\| stats sum(x) as s, avg(x) as v by a` | `summarize {s = sum(x), v = avg(x)}, by: {a}` | |
| `\| chart count over a by b` | `summarize count = count(), by: {a, b}` | DQL returns long format (one row per a,b pair). It has no pivot to wide columns. |
| `\| timechart span=5m count by a` | `makeTimeseries count = count(default: 0), by: {a}, interval: 5m` | Splunk shows 0 for empty buckets. DQL shows null unless you set `default: 0`. Splunk keeps the top 10 series plus OTHER. |
| `\| timechart span=1h avg(x)` | `makeTimeseries avg(x), interval: 1h` | Input must be numeric. |
| `\| bin _time span=1h \| stats count by _time` | `summarize count = count(), by: {t = bin(timestamp, 1h)}` | |
| `\| top limit=5 a` | `summarize count = count(), by: {a} \| sort count desc \| limit 5` | SPL also adds a `percent` column. See example 6. |
| `\| rare a` | `summarize count = count(), by: {a} \| sort count asc \| limit 10` | |
| `\| eventstats avg(x) as m by a` | `summarize` + `collectArray` + `expand`, or `lookup` against a second query | See example 9 and limitations. |
| `\| streamstats` | Usually no equivalent | See limitations. |
| `\| transaction k maxspan=30m` | `summarize` by `k` with min/max timestamp | Not 1:1. See example 10. |
| `\| delta x`, `\| accum x` | On a time series: `arrayDelta(x)`, `arrayCumulativeSum(x)` | Works on time buckets, not per event. |

## Sorting and limiting

| SPL | DQL | Watch out |
|---|---|---|
| `\| sort -a, b` | `sort a desc, b asc` | SPL `sort` without a count returns at most 10,000 rows. `sort 0` returns all. |
| `\| sort 20 -a` | `sort a desc \| limit 20` | |
| `\| head 20` (raw events) | `sort timestamp desc \| limit 20` | SPL returns the newest events first. In DQL, set the order explicitly. |
| `\| tail 20` | `sort timestamp asc \| limit 20` | |

## Combining searches

| SPL | DQL | Watch out |
|---|---|---|
| `\| join k [search ...]` | `join [ ... ], on: {k}` | Both default to inner. SPL takes 1 right-side match (`max=1`). DQL returns every match. Right-side fields get the prefix `right.`. |
| `\| join type=left k [...]` | `lookup [ ... ], sourceField: k, lookupField: k` | `lookup` keeps every left row and takes the first match, which is the closest to SPL. Fields get the prefix `lookup.`. |
| `\| join type=outer` | `join [ ... ], kind: leftOuter, on: {k}` | In SPL `outer` means left. DQL `kind: outer` is a full outer join. |
| `\| lookup file.csv k OUTPUT f` | `lookup [ load "/lookups/<name>" ], sourceField: k, lookupField: k, fields: {f}` | |
| `\| append [search ...]` | `append [ fetch ... ]` | |
| `\| appendcols` | No equivalent | Join on a shared key instead. |
| `field IN [subsearch]` | `filter field in [ fetch ... \| fields f ]` | The subquery must return a single field. |

## No DQL equivalent

These commands write data, send results, or run custom code. DQL only reads, so none of them can be translated: `outputlookup`, `collect`, `sendemail`, `script`, `map`, `foreach`, custom search commands, and `xyseries`/`untable`/`transpose` (DQL has no pivot). Use Dynatrace Workflows, OpenPipeline, or dashboard tiles for the job these commands did, and tell the user which part was dropped.

## Sources

- [DQL commands](https://docs.dynatrace.com/docs/platform/grail/dynatrace-query-language/commands)
- [DQL operators](https://docs.dynatrace.com/docs/platform/grail/dynatrace-query-language/operators)
- [DPL grammar](https://docs.dynatrace.com/docs/platform/grail/dynatrace-pattern-language/log-processing-grammar)
- [Splunk search commands](https://help.splunk.com/en/splunk-enterprise/search/spl-search-reference/10.0/search-commands/stats)
